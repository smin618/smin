# RAPA Ansible 자동화 — VMware vSphere

> 요구사항 8 (반복 작업 · 네트워크 작업 자동화) 구현 기록
> 대상 환경: VMware vSphere (vCenter + 분산 스위치 vDS)
> 최종 업데이트: 2026-06-27 · 현재 상태: **환경 구성 + 설치 완료**

---

## 1. 목표

손으로 하던 반복 작업과 네트워크 작업을 Ansible로 자동화한다.
특정 벤더·도구에 종속되지 않는 업계 표준 도구로 구성하고, 구성 드리프트(configuration drift)를 방지한다.

| 요구사항 | Ansible에서의 구현 |
|---|---|
| 네트워크 설정 한곳 일괄 관리 | Control node → vCenter API |
| 포트그룹·VLAN 표준 양식(템플릿) | 변수 목록 + `loop` + `community.vmware` |
| 망 추가 시 네트워크 설정 자동화 | Playbook 재실행 |
| OS 패치 등 반복 업무 | `apt` / `dnf` 모듈 (게스트 OS) |
| 구성 드리프트 방지 | 멱등성(idempotency) |
| 특정 도구·업체 비종속 | 오픈소스 + agentless |

---

## 2. 아키텍처

VMware 환경에서 Ansible은 ESXi/vCenter에 **SSH로 접속하지 않는다.**
Control node가 자기 자신(localhost)에서 모듈을 실행하고, 그 모듈이 vCenter API(HTTPS 443)를 호출한다.

```
[Control Node : WSL Ubuntu]
   │
   ├─ (API / HTTPS 443) ─► vCenter ─► dvSwitch · 포트그룹 · VLAN · VM   ← ① 인프라 평면
   │                                  (community.vmware, hosts: localhost)
   │
   └─ (SSH / WinRM) ───────► Guest VM (Linux/Windows) OS 패치·설정       ← ② 게스트 평면
                                      (일반 Ansible inventory)
```

---

## 3. 환경 / 사전 준비물

- **Control node**: WSL Ubuntu (Windows 위)
- **Python**: 3.12+ / venv (격리 환경)
- **Ansible**: community 패키지 + `community.vmware` collection
- **SDK**: pyvmomi 8.0.3+ (vSphere API Python SDK)
- **대상**: 운영 vCenter (분산 스위치 vDS) — 쓰기 연습은 격리 랩에서만
- **코드 저장소**: GitHub (smin618/smin)

---

## 4. 진행 기록

### Step 0 — 개념 정리 ✅
Ansible 자동화 = "절차(how)"를 시키는 게 아니라 "원하는 상태(what)"를 선언하면,
도구가 현재 상태를 점검해 목표와 다른 부분만 맞춰주는 **선언형(declarative)** 방식.
핵심 성질 3가지: **멱등성(idempotency)**, **agentless**, **IaC(코드로 관리되는 표준)**.

### Step 1 — Control node 준비 (WSL Ubuntu) ✅
Windows에 WSL Ubuntu 설치. 이것이 Ansible control node 역할을 한다.

### Step 2 — Git 설치 ✅
```bash
sudo apt update
sudo apt install -y git
git --version
```

### Step 3 — Ansible + VMware SDK 설치 (venv) ✅
```bash
sudo apt install -y python3-venv python3-pip
python3 -m venv ~/ansible-vmware
source ~/ansible-vmware/bin/activate      # 새 터미널 열 때마다 먼저 실행
pip install --upgrade pip
pip install ansible pyvmomi
ansible-galaxy collection install community.vmware
ansible --version
```

> 설치 방식은 **venv(pip) 한 길만** 사용한다.
> `apt install ansible-core` 와 혼용 금지 — ansible이 두 개 생겨 pyvmomi 인식이 충돌한다.

---

## 5. 핵심 결정 / 주의사항

- **설치는 venv 단일 경로**: VMware 모듈은 ansible을 실행하는 바로 그 Python에서 pyvmomi가 import 되어야 동작한다. apt `ansible-core` + 시스템 Python 조합은 PEP 668(externally-managed) 및 구버전 pyvmomi 문제로 회피.
- **운영 vCenter는 읽기 전용으로만 학습**: 전용 read-only 서비스 계정(`svc-ansible-ro@vsphere.local`)으로 접속해 info 모듈만 사용 → 운영 무영향. 권한 자체가 없어 실수로도 못 바꾼다.
- **쓰기 작업(포트그룹 생성·수정)은 격리 랩에서**: nested ESXi + VCSA(기본 60일 eval, vDS/HA/DRS 전부 동작) 또는 VMUG Advantage. 무료 ESXi는 API write가 막혀 있어 부적합.
- **자격증명은 `ansible-vault`로 암호화**: vCenter 비밀번호 평문 저장 금지.

---

## 6. 저장소 구조 (예정)

```
.
├── README.md
├── ansible.cfg
├── inventory/
├── group_vars/
│   └── all/
│       └── vault.yml          # ansible-vault 로 암호화
├── playbooks/
│   ├── check_vcenter.yml      # vCenter 연결 확인 (read-only)
│   ├── dvs_portgroup_info.yml # 운영 vDS 포트그룹 현황 조회
│   └── portgroups.yml         # (격리 랩) 포트그룹 표준 생성
└── roles/
```

---

## 7. 다음 단계

- [ ] WSL → vCenter 연결 확인 (`nc -zv <vcenter> 443`)
- [ ] vCenter에 read-only 서비스 계정(`svc-ansible-ro`) 생성
- [ ] `ansible-vault` 로 vCenter 자격증명 저장
- [ ] 첫 read-only playbook: `vmware_about_info` (연결 확인)
- [ ] `vmware_dvs_portgroup_info` 로 운영 vDS 포트그룹 현황 조회
- [ ] 동적 인벤토리(`vmware_vm_inventory`) 구성
- [ ] (격리 랩) `vmware_dvs_portgroup` 쓰기 + `loop` + 멱등성 검증
- [ ] 게스트 OS 패치 playbook (`apt` / `dnf`)
- [ ] Role 구조화 + Git 버전 관리 정착

---

## 8. 참고

- community.vmware: https://docs.ansible.com/ansible/latest/collections/community/vmware/
- pyvmomi: vSphere API Python SDK (Broadcom)
- VMware 모듈은 **API write 권한(유료 라이선스)** 필요 — 무료 ESXi 불가
