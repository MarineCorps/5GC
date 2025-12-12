# OAI CN5G SMF session_ambr 구현 분석

## 1. 개요

본 문서는 OpenAirInterface 5G Core Network의 Session Management Function(SMF)에서 **session_ambr (Session Aggregate Maximum Bit Rate)** 기능이 어떻게 구현되는지에 대한 상세 분석입니다.

---

## 2. session_ambr 데이터 구조

### 2.1 YAML 설정 파일에서의 정의

#### 파일: `/home/inho/5GC/docker-compose/conf/basic_nrf_config.yaml`
```yaml
smf:
  local_subscription_infos:
    - single_nssai: &embb_slice1
        sst: 1
        sd: "FFFFFF"
      dnn: "oai"
      qos_profile:
        5qi: 9
        session_ambr_ul: "200Mbps"    # UL (Uplink) 최대 비트레이트
        session_ambr_dl: "400Mbps"    # DL (Downlink) 최대 비트레이트
    
    - single_nssai: &embb_slice2
        sst: 1
        sd: "000001"
      dnn: "oai.ipv4"
      qos_profile:
        5qi: 9
        session_ambr_ul: "100Mbps"
        session_ambr_dl: "200Mbps"
    
    - single_nssai: &custom_slice
        sst: 2
        sd: "000002"
      dnn: "default"
      qos_profile:
        5qi: 9
        session_ambr_ul: "50Mbps"
        session_ambr_dl: "100Mbps"
```

**설정 위치:**
- 기본: `/home/inho/5GC/docker-compose/conf/` 의 YAML 설정 파일들
- Helm Charts: `/home/inho/5GC/charts/e2e_scenarios/case*/config.yaml`
- 테스트: `/home/inho/5GC/test/template/template_config.yaml`

### 2.2 C++ 데이터 구조 정의

#### 파일: `src/smf_app/smf_context.hpp` (GitLab)
```cpp
// 주요 include 파일
#include "SessionAmbr.hpp"
#include "3gpp_29.244.h"

// SMF Context 클래스에서 세션 AMBR 처리
class smf_context {
public:
    /*
     * Get the default value of Session-AMBR (NAS format)
     * @param [oai::nas::SessionAmbr &] session_ambr
     * @param [const snssai_t &] snssai
     * @param [const std::string &] dnn
     * @return void
     */
    void get_session_ambr(
        oai::nas::SessionAmbr& session_ambr, 
        const snssai_t& snssai,
        const std::string& dnn
    );

    /*
     * Get the default value of Session-AMBR (PFCP format)
     * @param [session_ambr_t &] session_ambr
     * @param [const snssai_t &] snssai
     * @param [const std::string &] dnn
     * @return void
     */
    void get_session_ambr(
        session_ambr_t& session_ambr, 
        const snssai_t& snssai,
        const std::string& dnn
    );
};
```

### 2.3 세션 AMBR 데이터 타입

#### NAS 형식 (`SessionAmbr`)
- NAS 메시지에서 UE로 전송되는 형식
- 압축된 대역폭 값 사용
- UE가 세션의 최대 처리 속도를 인식하도록 함

#### PFCP 형식 (`session_ambr_t`)
- N4 인터페이스(SMF-UPF)를 통해 전달되는 형식
- 3GPP TS 29.244 표준 준수
- UPF에서 실제 패킷 처리에 사용

---

## 3. YAML 설정 파일 파싱

### 3.1 파싱 흐름

```
YAML 설정 파일 (config.yaml)
    ↓
SMF 부팅 시 설정 로더
    ↓
local_subscription_infos 파싱
    ↓
QoS 프로필 추출
    ↓
session_ambr_ul, session_ambr_dl 값 추출
    ↓
dnn_configurations 맵에 저장
    ↓
PDU Session 생성 시 참조
```

### 3.2 파싱 관련 주요 구조

#### 데이터베이스 스키마
파일: `/home/inho/5GC/charts/oai-5g-core/mysql/initialization/oai_db-mini.sql`

```sql
-- UE AMBR (전체 사용자 집계 최대 비트레이트)
CREATE TABLE `subscription_data` (
  `ue_ambr_ul` bigint(20) unsigned DEFAULT '50000000' 
    COMMENT 'Maximum Aggregated uplink MBRs across all Non-GBR bearers',
  `ue_ambr_dl` bigint(20) unsigned DEFAULT '100000000' 
    COMMENT 'Maximum Aggregated downlink MBRs across all Non-GBR bearers',
  `aggregate_ambr_ul` int(10) unsigned DEFAULT '50000000',
  `aggregate_ambr_dl` int(10) unsigned DEFAULT '100000000'
);
```

### 3.3 설정 값 예시

| 시나리오 | session_ambr_ul | session_ambr_dl | DNN | 용도 |
|---------|-----------------|-----------------|-----|------|
| eMBB Slice 1 | 200Mbps | 400Mbps | oai | 광대역 모바일 브로드밴드 |
| eMBB Slice 2 | 100Mbps | 200Mbps | oai.ipv4 | 표준 브로드밴드 |
| Custom | 50Mbps | 100Mbps | default | 기본 서비스 |
| 부하 테스트 | 1000Mbps | 1000Mbps | - | 테스트용 고속 |

---

## 4. PFCP N4 인터페이스를 통한 UPF 전달

### 4.1 PFCP 메시지 흐름

```
SMF (Session Management)
    │
    └─→ PFCP Session Establishment Request
            │
            ├─ Session AMBR (UL)
            ├─ Session AMBR (DL)
            ├─ PDRs (Packet Detection Rules)
            ├─ FARs (Forwarding Action Rules)
            └─ URRs (Usage Report Rules)
    │
    └─← PFCP Session Establishment Response
            │
            └─ F-TEID (Forwarding Tunnel Endpoint Identifier)

        UPF (User Plane Function)
```

### 4.2 PFCP 메시지 구성 (3GPP TS 29.244)

#### Session Establishment 메시지 구조
```
PFCP Session Establishment Request
├─ Node ID
├─ CP F-SEID (Control Plane F-SEID)
├─ PDR (Packet Detection Rule)
│   ├─ PDR ID
│   ├─ Precedence
│   ├─ PFD (Protocol Filter Definition)
│   └─ Outer Header Removal
├─ FAR (Forwarding Action Rule)
│   ├─ FAR ID
│   ├─ Apply Action
│   ├─ Forwarding Parameters
│   └─ Duplicating Parameters
├─ QER (QoS Enforcement Rule) ← session_ambr 정보
│   ├─ QER ID
│   ├─ QoS Flow ID (QFI)
│   ├─ Gate Status (UL/DL)
│   ├─ MBR (Maximum Bit Rate) ← UL/DL AMBR
│   ├─ GBR (Guaranteed Bit Rate)
│   └─ Packet Rate Status
├─ URR (Usage Report Rule)
│   ├─ URR ID
│   ├─ Measurement Method
│   ├─ Reporting Triggers
│   └─ Usage Information
└─ Created PDR
```

### 4.3 Session AMBR의 PFCP 전송 방식

#### UPF에 전달되는 AMBR 값

**file: 설정에서 추출된 값**
```
session_ambr_ul: 200Mbps → 변환 → 200,000,000 bps (UPF에서 사용)
session_ambr_dl: 400Mbps → 변환 → 400,000,000 bps (UPF에서 사용)
```

#### 3GPP TS 29.244 표준의 AMBR 인코딩

**Bit Rate Unit**
- 8 bits: 64 kbps 단위로 인코딩
- 예: 200 Mbps = 200,000,000 / 64,000 = 3,125 (최적 단위 선택)

### 4.4 실제 구현 관련 파일 (GitLab)

#### Repository: `oai-cn5g-smf`

**주요 파일 경로:**
```
src/smf_app/
├── smf_context.hpp          # session_ambr 함수 선언
├── smf_context.cpp          # session_ambr 함수 구현
├── smf_pfcp_association.hpp # PFCP 세션 관리
├── smf_pfcp_association.cpp # PFCP 세션 구현
└── SessionAmbr.hpp          # session_ambr 클래스 정의

src/pfcp/
├── msg_pfcp.hpp             # PFCP 메시지 구조
├── pfcp_session.hpp         # PFCP 세션
└── pfcp_session.cpp         # PFCP 세션 구현

src/common/
├── 3gpp_29.244.h           # PFCP 헤더 (3GPP TS 29.244)
├── 3gpp_29.502.h           # SMF NAS 규격
└── 3gpp_29.244.hpp         # PFCP C++ 래퍼
```

---

## 5. Rate Limiting 및 Traffic Shaping 구현

### 5.1 UPF에서의 Rate Limiting 메커니즘

#### 5.1.1 QER (QoS Enforcement Rule) 기반

```
SMF가 PFCP 세션 설정 시 UPF에 전달:
├─ QER (QoS Enforcement Rule)
│   ├─ QFI (QoS Flow ID): 각 QoS 플로우별 AMBR 적용
│   ├─ MBR (Maximum Bit Rate): UL/DL 제한 대역폭
│   │   ├─ MBR_UL: session_ambr_ul 값 적용
│   │   └─ MBR_DL: session_ambr_dl 값 적용
│   ├─ Gate Status: 트래픽 허용/거부
│   │   ├─ UL Gate: UL 트래픽 제어
│   │   └─ DL Gate: DL 트래픽 제어
│   └─ Packet Rate Status: 패킷 레이트 제한

UPF의 각 QFI별 처리:
├─ Token Bucket 또는 Traffic Shaper 사용
├─ 수신 패킷의 대역폭 계산
├─ MBR 초과 시 처리:
│   ├─ 옵션 1: 패킷 드롭 (Drop)
│   ├─ 옵션 2: 트래픽 마킹 (Marking)
│   └─ 옵션 3: 버퍼링 (Buffering)
└─ 사용량 통계 기록
```

### 5.1.2 OAI UPF의 구현 (VPP 기반)

#### Repository: `oai-cn5g-upf-vpp`

**구현 메커니즘:**

```c
// VPP (Vector Packet Processing) 사용
// - 고성능 데이터 플레인 구현
// - DPDK 기반 패킷 처리

// 주요 처리 함수:
1. pfcp_session_create()
   └─ QER 파싱 및 저장
   └─ MBR 값 추출 (session_ambr)
   └─ Traffic Shaper 설정

2. packet_processing()
   ├─ QFI 추출 (GTP-U 헤더에서)
   ├─ QER 룩업
   ├─ MBR 체크
   ├─ Rate Limiter 적용
   └─ 패킷 포워딩/드롭

3. rate_limiter_algorithm()
   ├─ Token Bucket Algorithm
   │   ├─ 토큰 누적 (초당 MBR)
   │   ├─ 패킷 전송 시 토큰 소비
   │   └─ 토큰 부족 시 드롭
   │
   └─ Leaky Bucket Algorithm
       ├─ 고정 크기 버킷
       ├─ 패킷 큐잉
       └─ 초과 패킷 드롭
```

### 5.1.3 BPF (Berkeley Packet Filter) 기반 구현

#### Repository: `oai-cn5g-upf` (eBPF variant)

```c
// eBPF (Extended BPF) 를 이용한 커널 공간 처리

// bpf_prog_rate_limiting():
// - 커널에서 직접 패킷 처리
// - 매우 낮은 지연시간
// - 고처리량 rate limiting

struct qer_mbr {
    uint64_t mbr_ul;  // Uplink MBR (bps)
    uint64_t mbr_dl;  // Downlink MBR (bps)
    uint64_t tokens_ul;  // 남은 토큰
    uint64_t tokens_dl;  // 남은 토큰
    uint64_t last_update; // 마지막 업데이트 타임스탐프
};

// 초당 처리:
1. 각 QFI의 session_ambr_ul/dl 값으로 토큰 재충전
2. 수신 패킷의 크기로 토큰 소비
3. 토큰 부족 시 패킷 드롭
```

### 5.2 실제 Rate Limiting 구현 옵션

#### Configuration 옵션 (YAML)

```yaml
upf:
  support_features:
    enable_bpf_datapath: no    # BPF 기반 처리 (고속)
                               # no: Simple Switch 기반 처리
    enable_snat: yes           # Source NAT 활성화
  
  # Rate Limiting 관련 설정
  qer_config:
    use_token_bucket: yes      # Token Bucket 알고리즘 사용
    bucket_size: 1024          # 버킷 크기 (패킷 수)
    rate_update_interval: 10ms # 토큰 재충전 주기
```

### 5.3 모니터링 및 통계

#### URR (Usage Report Rule)

```
SMF가 설정한 URR:
├─ 목적: Session 트래픽 통계 수집
├─ 수집 정보:
│   ├─ 업링크 트래픽 (bytes/packets)
│   ├─ 다운링크 트래픽 (bytes/packets)
│   ├─ MBR 초과 패킷 수
│   ├─ 드롭된 패킷 수
│   └─ 처리 시간 분포
│
└─ 보고 트리거:
    ├─ 주기적 보고 (일정 시간 간격)
    ├─ Volume 기반 보고 (일정 데이터량 이상)
    ├─ 세션 해제 시 보고
    └─ AMBR 초과 시 보고
```

---

## 6. 코드 파일 정리 및 함수명

### 6.1 주요 파일 및 함수 매핑

#### GitLab 주소: `https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-smf`

| 파일 경로 | 함수명 | 기능 | 라인 수 |
|----------|--------|------|--------|
| `src/smf_app/smf_context.hpp` | `get_session_ambr()` | session_ambr 조회 (NAS format) | 선언부 |
| `src/smf_app/smf_context.hpp` | `get_session_ambr()` (overload) | session_ambr 조회 (PFCP format) | 선언부 |
| `src/smf_app/smf_context.cpp` | `get_session_ambr()` | session_ambr 구현 | 구현부 |
| `src/smf_app/SessionAmbr.hpp` | `SessionAmbr::encode()` | NAS 메시지 인코딩 | 클래스 메서드 |
| `src/smf_app/SessionAmbr.hpp` | `SessionAmbr::decode()` | NAS 메시지 디코딩 | 클래스 메서드 |
| `src/pfcp/msg_pfcp.hpp` | `session_ambr_t` | PFCP 세션 AMBR 구조체 | 정의 |
| `src/pfcp/pfcp_session.cpp` | `create_session()` | PFCP 세션 생성 및 AMBR 설정 | 구현부 |
| `src/smf_app/smf_n4.cpp` | `send_pfcp_session_establishment_request()` | UPF에 PFCP 메시지 전송 | 구현부 |
| `src/smf_app/smf_pfcp_association.cpp` | `establish_pfcp_association()` | PFCP 연결 설정 | 구현부 |
| `src/common/3gpp_29.244.hpp` | 3GPP PFCP 구조들 | PFCP 표준 구현 | 헤더 |

### 6.2 설정 파일 파싱 관련

| 파일 경로 | 기능 | 위치 |
|----------|------|------|
| `config.yaml` | YAML 설정 로드 | 시작 시 |
| `smf_config.cpp` | YAML 파서 | src/smf_app/ |
| `dnn_configuration_t` | DNN별 설정 저장 | src/smf_app/smf_context.hpp |
| `qos_profile_t` | QoS 프로필 구조 | src/common/ |

### 6.3 데이터베이스 관련

| 파일 경로 | 기능 | 설명 |
|----------|------|------|
| `oai_db-mini.sql` | DB 스키마 | UE AMBR, Aggregate AMBR |
| `subscription_data` 테이블 | UE 구독 정보 저장 | UDR에서 조회 |

---

## 7. 세션 AMBR 적용 흐름도

```
┌─────────────────────────────────────────────────────────────┐
│                    UE 및 gNB 요청                            │
│          (PDU Session Establishment Request)                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────┐
│                    AMF → SMF (N11)                          │
│          CreateSMContextRequest 수신                         │
│          - DNN, SNSSAI 정보 포함                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────┐
│              SMF Context 조회                               │
│  get_session_ambr(snssai, dnn)                             │
│  - local_subscription_infos 검색                           │
│  - 해당 DNN/SNSSAI의 session_ambr_ul/dl 추출              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ├─────────────────────┐
                     │                     │
                     ↓                     ↓
        ┌──────────────────┐    ┌────────────────┐
        │   NAS 메시지      │    │ PFCP 메시지    │
        │  (UE로 전송)      │    │ (UPF로 전송)   │
        │                  │    │                │
        │ SessionAmbr      │    │ session_ambr_t │
        │ (압축 형식)       │    │ (3GPP 29.244)  │
        └────────┬─────────┘    └────────┬────────┘
                 │                       │
                 ↓                       ↓
        ┌──────────────────┐    ┌────────────────┐
        │    GNB로 송신     │    │   UPF로 송신   │
        │   (RRC 메시지)    │    │ (N4 인터페이스)│
        │                  │    │ Session Est.   │
        │ - AMBR 정보       │    │ Request        │
        │ - QoS 정보       │    │                │
        └────────┬─────────┘    └────────┬────────┘
                 │                       │
                 │                       ↓
                 │            ┌────────────────┐
                 │            │   UPF 처리      │
                 │            │                │
                 │            │ QER 설정       │
                 │            │ ├─ MBR_UL      │
                 │            │ ├─ MBR_DL      │
                 │            │ └─ Rate Limiter│
                 │            │                │
                 │            └────────┬────────┘
                 │                     │
                 ↓                     ↓
        ┌──────────────────┐    ┌────────────────┐
        │   UE 처리        │    │ 트래픽 처리    │
        │                 │    │                │
        │ Session 생성    │    │ Token Bucket   │
        │ AMBR 적용       │    │ or BPF 기반    │
        │ QoS 관리        │    │ Rate Limiting  │
        │ 패킷 전송       │    │ 패킷 드롭/전송 │
        └──────────────────┘    └────────────────┘
```

---

## 8. 실제 구현 예시 (의사 코드)

### 8.1 SMF에서의 session_ambr 추출

```cpp
// src/smf_app/smf_context.cpp
void smf_context::get_session_ambr(
    session_ambr_t& session_ambr, 
    const snssai_t& snssai,
    const std::string& dnn) 
{
    // 1. DNN 구독 정보 검색
    std::shared_ptr<session_management_subscription> ss;
    if (find_dnn_subscription(snssai, ss)) {
        // 2. DNN 설정 조회
        std::shared_ptr<dnn_configuration_t> dnn_config;
        ss->find_dnn_configuration(dnn, dnn_config);
        
        if (dnn_config) {
            // 3. QoS 프로필에서 session_ambr 추출
            session_ambr.ambr_ul = dnn_config->qos_profile.session_ambr_ul;
            session_ambr.ambr_dl = dnn_config->qos_profile.session_ambr_dl;
            
            // 예: "200Mbps" → 200000000 bps
            // 예: "100Mbps" → 100000000 bps
        }
    }
}
```

### 8.2 PFCP 세션 생성 시 AMBR 설정

```cpp
// src/pfcp/pfcp_session.cpp
void pfcp_session::create_session(
    const session_ambr_t& session_ambr,
    const std::vector<qer_t>& qers)
{
    // 1. QER 생성 및 AMBR 설정
    for (const auto& qer : qers) {
        pfcp::qer_t qer_msg;
        qer_msg.qer_id = qer.id;
        qer_msg.qfi = qer.qfi;
        
        // 2. session_ambr 값을 MBR로 설정
        qer_msg.mbr.ul_mbr = session_ambr.ambr_ul;
        qer_msg.mbr.dl_mbr = session_ambr.ambr_dl;
        
        // 3. Gate Status 설정 (초기값: 열림)
        qer_msg.gate_status.ul_gate = GATE_OPEN;
        qer_msg.gate_status.dl_gate = GATE_OPEN;
        
        // 4. Rate Limiter 파라미터 설정
        qer_msg.rate_limiter.tokens_ul = session_ambr.ambr_ul / 8000; // ms당 바이트
        qer_msg.rate_limiter.tokens_dl = session_ambr.ambr_dl / 8000;
        
        qers_list.push_back(qer_msg);
    }
}
```

### 8.3 PFCP 메시지 전송

```cpp
// src/smf_app/smf_n4.cpp
bool smf_n4::send_pfcp_session_establishment_request(
    const std::string& upf_host,
    const session_ambr_t& session_ambr,
    const std::vector<qer_t>& qers)
{
    // 1. PFCP 메시지 생성
    pfcp::pfcp_session_establishment_request req;
    req.node_id = smf_node_id;
    req.cp_fseid = current_cp_fseid;
    
    // 2. QER 추가 (session_ambr 포함)
    for (const auto& qer : qers) {
        req.add_qer(qer);
    }
    
    // 3. PFCP 메시지 인코딩
    std::vector<uint8_t> encoded_msg = req.encode();
    
    // 4. UPF로 전송 (N4 인터페이스)
    return send_to_upf(upf_host, N4_PORT, encoded_msg);
}
```

### 8.4 UPF에서의 Rate Limiting (의사 코드)

```cpp
// oai-cn5g-upf / oai-cn5g-upf-vpp
void upf_packet_processing(const packet_t& pkt)
{
    // 1. GTP-U 헤더에서 QFI 추출
    uint8_t qfi = extract_qfi_from_gtp_header(pkt);
    
    // 2. QER 검색
    qer_t* qer = find_qer_by_qfi(qfi);
    if (!qer) return;  // QER 없으면 드롭
    
    // 3. MBR 체크 (Token Bucket Algorithm)
    uint64_t pkt_size = pkt.length;
    uint64_t current_time = get_current_time_us();
    
    // 토큰 재충전 (시간 경과량 기반)
    uint64_t elapsed = current_time - qer->last_update_time;
    uint64_t tokens_generated = (qer->mbr_ul * elapsed) / 1_000_000;  // μs → rate 변환
    
    qer->tokens_ul = std::min(
        qer->tokens_ul + tokens_generated,
        qer->mbr_ul / 8  // 최대 1/8초치 누적
    );
    qer->last_update_time = current_time;
    
    // 4. 패킷 전송 또는 드롭 결정
    if (qer->tokens_ul >= pkt_size) {
        // 토큰 충분 → 패킷 전송
        qer->tokens_ul -= pkt_size;
        forward_packet(pkt);
        qer->stats.bytes_forwarded += pkt_size;
    } else {
        // 토큰 부족 → 패킷 드롭
        drop_packet(pkt);
        qer->stats.bytes_dropped += pkt_size;
        qer->stats.packets_dropped++;
    }
}
```

---

## 9. 주요 설정 시나리오

### 9.1 eMBB (Enhanced Mobile Broadband) 슬라이스

```yaml
slices:
  - sst: 1
    sd: "FFFFFF"
    dnns:
      - name: "oai"
        session_ambr_ul: "200Mbps"  # 고속 업로드
        session_ambr_dl: "400Mbps"  # 고속 다운로드
    use_case: "Video Streaming, Large File Transfer"
```

**특징:**
- 높은 대역폭 할당
- 낮은 지연시간 요구 X
- 처리량(Throughput) 중심

### 9.2 URLLC (Ultra-Reliable Low-Latency) 슬라이스

```yaml
slices:
  - sst: 3
    sd: "000003"
    dnns:
      - name: "critical"
        session_ambr_ul: "10Mbps"   # 낮은 대역폭
        session_ambr_dl: "10Mbps"
    use_case: "Industrial IoT, Vehicle Control"
```

**특징:**
- 낮은 대역폭
- 극도로 낮은 지연시간 요구
- 신뢰성 중심

### 9.3 mMTC (Massive Machine-Type Communication) 슬라이스

```yaml
slices:
  - sst: 2
    sd: "000002"
    dnns:
      - name: "iot"
        session_ambr_ul: "1Mbps"    # 매우 낮은 대역폭
        session_ambr_dl: "2Mbps"
    use_case: "Sensor Networks, Smart Metering"
```

**특징:**
- 매우 낮은 대역폭
- 많은 동시 연결
- 비용 효율성 중심

---

## 10. 관련 3GPP 표준

| 표준 | 내용 | 관련성 |
|------|------|--------|
| 3GPP TS 29.244 | PFCP (Packet Forwarding Control Protocol) | session_ambr의 N4 전송 방식 |
| 3GPP TS 29.502 | SMF 서비스 기반 인터페이스 | session_ambr의 NAS 메시지 포함 |
| 3GPP TS 24.501 | NAS 5GSM (5G Session Management) | session_ambr의 UE 전송 형식 |
| 3GPP TS 38.415 | PDCP for NR | session_ambr 기반 QoS 적용 |

---

## 11. 문제 해결 및 모니터링

### 11.1 일반적인 문제

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| 트래픽 제한 안 됨 | UPF의 QER 미설정 | PFCP 메시지 확인, UPF 로그 검사 |
| 예상보다 낮은 처리량 | AMBR 값 과소 설정 | 설정 파일 검증, DNN 구독 정보 확인 |
| 특정 DNN만 제한 안 됨 | 선택적 설정 오류 | YAML 구문 검증, SMF 재시작 |
| Rate limiting 지연 | 버킷 크기 작음 | URR 간격 조정, 버킷 크기 증가 |

### 11.2 모니터링 포인트

```bash
# SMF 로그 확인
docker logs oai-smf | grep -i "session_ambr\|AMBR\|QER"

# UPF 통계 확인
docker exec oai-upf vppctl show interface
docker exec oai-upf vppctl show error

# PFCP 메시지 캡처
tcpdump -i eth0 -n udp port 8805

# 데이터베이스에서 구독 정보 확인
SELECT subscription_data.*, dnn_configurations.* 
FROM subscription_data 
JOIN dnn_configurations USING (imsi)
WHERE dnn_configurations.dnn = 'oai';
```

---

## 12. 결론

### 12.1 Session AMBR 구현 요약

**5단계 구현 흐름:**
1. **YAML 설정**: `session_ambr_ul`, `session_ambr_dl` 정의
2. **SMF 파싱**: 설정 파일에서 값 추출 및 메모리 저장
3. **NAS 인코딩**: UE에게 session_ambr 정보 전송 (SessionAmbr 클래스)
4. **PFCP 전송**: UPF에게 MBR 정보 전달 (PFCP 메시지)
5. **UPF 적용**: QER 기반 rate limiting 실행 (Token Bucket 알고리즘)

### 12.2 핵심 파일 (GitLab)

```
https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-smf

주요 파일:
├── src/smf_app/smf_context.hpp        (session_ambr 함수 선언)
├── src/smf_app/smf_context.cpp        (session_ambr 함수 구현)
├── src/smf_app/SessionAmbr.hpp        (NAS 메시지 인코딩/디코딩)
├── src/pfcp/msg_pfcp.hpp              (PFCP 메시지 구조)
├── src/pfcp/pfcp_session.cpp          (PFCP 세션 생성)
├── src/smf_app/smf_n4.cpp             (PFCP 메시지 전송)
└── src/common/3gpp_29.244.hpp         (3GPP TS 29.244 구현)
```

### 12.3 Rate Limiting 구현

- **표준**: 3GPP TS 29.244 (PFCP 프로토콜)
- **알고리즘**: Token Bucket (초당 AMBR 값을 토큰으로 변환)
- **구현 위치**: UPF (VPP 또는 eBPF)
- **제어 메커니즘**: QER (QoS Enforcement Rule)

---

## 부록: 파일 참조

### 부록 A. 설정 파일 위치

```
5GC/
├── docker-compose/conf/
│   ├── basic_nrf_config.yaml          ✓ session_ambr 설정
│   └── slicing_base_config.yaml
├── charts/e2e_scenarios/
│   ├── case1/config.yaml              ✓ session_ambr 설정
│   ├── case2/config.yaml              ✓ session_ambr 설정
│   └── case3/config.yaml              ✓ session_ambr 설정
├── charts/oai-5g-core/
│   ├── oai-5g-basic/config.yaml       ✓ session_ambr 설정
│   ├── oai-5g-advance/config.yaml     ✓ session_ambr 설정
│   ├── oai-5g-mini/config.yaml        ✓ session_ambr 설정
│   ├── oai-smf/config.yaml            ✓ session_ambr 설정
│   ├── mysql/initialization/
│   │   └── oai_db-mini.sql            ✓ AMBR DB 스키마
│   └── oai-lmf/config.yaml            ✓ session_ambr 설정
├── test/
│   ├── json_templates/
│   │   └── smf_json_strings.py        ✓ session_ambr JSON
│   └── template/
│       └── template_config.yaml       ✓ session_ambr 설정
└── ci-scripts/charts/
    └── oai-5g-basic/config.yaml       ✓ session_ambr 설정 (1000Mbps)
```

### 부록 B. 관련 링크

- **OAI GitLab**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-smf
- **3GPP TS 29.244**: PFCP Protocol Standard
- **3GPP TS 29.502**: SMF Service-Based Interface
- **3GPP TS 24.501**: NAS 5GSM Messages

---

**작성일**: 2025-12-12  
**분석 대상**: OpenAirInterface 5G Core Network (OAI CN5G) v2.2.0  
**SMF Repository**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-smf
