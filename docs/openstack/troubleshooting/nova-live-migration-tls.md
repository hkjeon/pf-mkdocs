# Live Migration 실패 (대용량 메모리 VM)

!!! info "환경"
    - OpenStack: OSA 배포 환경
    - 대상: 메모리 32GB 이상 VM
    - 확인 시점: 2026-01

## 증상

- 대용량 메모리 VM Live-Migration 수행 시 실패
- 마이그레이션 실패 후 다른 VM 볼륨 경로가 강제로 끊기는 2차 영향 발생

??? failure "커널 로그"
```text
    traps: return path[1838007] general protection fault
    ip:7f5caa29039e sp:7f1c7b1f00c0 error:0
    in libgnutls.so.30.31.0[7f5caa284000+129000]
```

## 원인

| 구분 | 내용 |
|---|---|
| 근본 원인 | TLS 1.3 사용 시 libgnutls의 Re-Key 처리 결함 |
| 패치 상태 | 미병합 (2026-01 기준) |
| 2차 영향 | Error 상태 VM 강제 Active 전환 후 재기동 시 타 VM 볼륨 점유 |

!!! warning "적용 판단 기준"
    메모리 사용량이 높고 VM 메모리가 32GB 이상인 경우, 현 시점에서는 TCP 방식 적용이 사실상 필수.

## 조치

Live Migration 전송 방식을 TLS에서 TCP로 변경.

=== "수동 적용"

    **1. libvirtd 설정** — `/etc/libvirt/libvirtd.conf`

```diff
    - listen_tls = 1
    - listen_tcp = 0
    + listen_tls = 0
    + listen_tcp = 1
    + auth_tcp = "none"
```

    **2. nova 설정** — `nova.conf`

```ini
    live_migration_with_native_tls = False
    live_migration_tunnelled = False
    live_migration_scheme = tcp
```

    **3. libvirtd 소켓 전환 및 재기동**

```bash
    systemctl stop libvirtd
    systemctl stop libvirtd.socket

    systemctl disable libvirtd-tls.socket --now
    systemctl enable libvirtd-tcp.socket --now

    systemctl start libvirtd
```

    **4. nova-compute 재기동**

```bash
    systemctl restart nova-compute
```

=== "OSA 적용"

    `user_variables.yml`

```yaml
    nova_libvirtd_listen_tls: 0
    nova_libvirtd_listen_tcp: 1
    nova_libvirtd_auth_tcp: none
    nova_qemu_vnc_tls: 0

    nova_nova_conf_overrides:
      libvirt:
        live_migration_permit_auto_converge: True
        live_migration_permit_post_copy: False
        live_migration_with_native_tls: False
        live_migration_uri: {}
        live_migration_tunnelled: False
        live_migration_scheme: tcp
```

## 검증

부하 상태에서 마이그레이션 안정성을 확인.

```bash
# 메모리 cache/buffer 50GB 점유
fallocate -l 50G /var/tmp/cachefile
dd if=/var/tmp/cachefile of=/dev/null bs=8M status=progress &

# vCPU 3개 사용, 가용 메모리 90% 점유 (60분)
stress-ng --vm 3 --vm-bytes 90% --vm-keep --vm-populate \
          --timeout 60m --metrics-brief &
```

```bash
systemctl status libvirtd
systemctl status nova-compute
```

!!! success "시험 결과"
    | 환경 | 구성 | 결과 |
    |---|---|---|
    | PoC 장비 | 256GB VM | 정상 |
    | 운영 장비 | 64GB + 256GB 혼재 | 정상 |
    | Guest OS | Windows | 정상 |

## 참고

| 구분 | 링크 |
|---|---|
| OSA 문의 | [Launchpad #823430](https://answers.launchpad.net/openstack-ansible/+question/823430) |
| Nova 버그 | [Launchpad #2133501](https://bugs.launchpad.net/nova/+bug/2133501) |
| QEMU 이슈 | [GitLab #1937](https://gitlab.com/qemu-project/qemu/-/issues/1937) |
| Nova 패치 | [Gerrit 955784](https://review.opendev.org/c/openstack/nova/+/955784) |
| libvirt 버그 | [Launchpad #2133183](https://bugs.launchpad.net/ubuntu/+source/libvirt/+bug/2133183) |
