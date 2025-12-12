# OAI CN5G SMF session_ambr 분석 - 최종 요약

## 빠른 참조 가이드 (Quick Reference)

### 1️⃣ session_ambr 정의 위치

| 레이어 | 파일 | 변수명 | 형식 |
|--------|------|--------|------|
| **설정** | `docker-compose/conf/*.yaml` | `session_ambr_ul`, `session_ambr_dl` | String (e.g., "200Mbps") |
| **NAS** | `src/smf_app/SessionAmbr.hpp` | `SessionAmbr` | 클래스 (압축 형식) |
| **PFCP** | `src/pfcp/msg_pfcp.hpp` | `session_ambr_t` | 구조체 (3GPP TS 29.244) |
| **데이터베이스** | `oai_db-mini.sql` | `ue_ambr_ul`, `ue_ambr_dl` | BIGINT (bps) |

---

### 2️⃣ 5단계 구현 흐름

```
┌─────────────────────────────────────────────────────────┐
│ 1단계: YAML 설정 파일에서 값 정의                        │
│  session_ambr_ul: "200Mbps"                             │
│  session_ambr_dl: "400Mbps"                             │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│ 2단계: SMF 시작 시 YAML 파싱                            │
│  dnn_configuration_t에 저장                              │
│  파일: src/smf_app/smf_config.cpp                       │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│ 3단계: PDU Session 생성 시 session_ambr 조회           │
│  get_session_ambr(snssai, dnn)                         │
│  파일: src/smf_app/smf_context.cpp                     │
└──────────────────┬──────────────────────────────────────┘
                   │
       ┌───────────┴──────────┐
       │                      │
┌──────▼──────────┐  ┌───────▼────────────┐
│ NAS 메시지      │  │ PFCP 메시지        │
│ (UE로 송신)     │  │ (UPF로 송신)       │
│                │  │                   │
│ SessionAmbr     │  │ session_ambr_t     │
│ 인코드          │  │ 인코드 (3GPP TS)   │
└──────┬──────────┘  └────────┬──────────┘
       │                      │
┌──────▼──────────┐  ┌───────▼────────────┐
│ 4단계: UE        │  │ 5단계: UPF         │
│ 세션 AMBR 인식  │  │ Rate Limiting 적용 │
│ QoS 적용        │  │ Token Bucket       │
└─────────────────┘  │ eBPF 기반          │
                     └────────────────────┘
```

---

### 3️⃣ 핵심 파일 5개

#### 📄 파일 1: `src/smf_app/smf_context.hpp`
```cpp
// session_ambr 함수 선언 위치
class smf_context {
public:
    void get_session_ambr(
        oai::nas::SessionAmbr& session_ambr,
        const snssai_t& snssai,
        const std::string& dnn
    );
    
    void get_session_ambr(
        session_ambr_t& session_ambr,
        const snssai_t& snssai,
        const std::string& dnn
    );
};
```

#### 📄 파일 2: `src/smf_app/smf_context.cpp`
```cpp
// session_ambr 함수 구현
// 동작:
// 1. dnn_subscriptions 맵에서 snssai로 구독 정보 검색
// 2. session_management_subscription에서 dnn 설정 검색
// 3. dnn_configuration_t에서 qos_profile 추출
// 4. session_ambr_ul/dl 값 반환
```

#### 📄 파일 3: `src/smf_app/SessionAmbr.hpp`
```cpp
// NAS 형식의 session_ambr 클래스
class SessionAmbr {
public:
    void encode();   // NAS 메시지 형식으로 인코딩
    void decode();   // NAS 메시지에서 디코딩
    void set_ul_ambr(uint64_t ambr);
    void set_dl_ambr(uint64_t ambr);
};
```

#### 📄 파일 4: `src/pfcp/msg_pfcp.hpp`
```cpp
// PFCP 형식의 session_ambr 구조체
struct session_ambr_t {
    uint64_t ambr_ul;    // Uplink AMBR (bps)
    uint64_t ambr_dl;    // Downlink AMBR (bps)
    
    void encode();       // PFCP 메시지로 인코딩
    void decode();       // PFCP 메시지에서 디코딩
};
```

#### 📄 파일 5: `src/smf_app/smf_n4.cpp`
```cpp
// PFCP 메시지를 UPF로 전송
bool send_pfcp_session_establishment_request(
    const std::string& upf_host,
    const session_ambr_t& session_ambr,
    const std::vector<qer_t>& qers
) {
    // QER에 session_ambr 값을 MBR로 설정
    // PFCP 메시지 인코딩
    // UPF로 전송 (UDP 8805)
}
```

---

### 4️⃣ PFCP 메시지 구조

```
PFCP Session Establishment Request
│
├─ QER (QoS Enforcement Rule)
│   ├─ QER ID: 식별자
│   ├─ QFI (QoS Flow ID): 0~63
│   ├─ MBR (Maximum Bit Rate) ← session_ambr 값 사용
│   │   ├─ MBR_UL: session_ambr_ul 값
│   │   └─ MBR_DL: session_ambr_dl 값
│   ├─ Gate Status
│   │   ├─ UL Gate: OPEN/CLOSED
│   │   └─ DL Gate: OPEN/CLOSED
│   └─ Packet Rate Status
│
├─ PDR (Packet Detection Rule)
├─ FAR (Forwarding Action Rule)
└─ URR (Usage Report Rule)
```

---

### 5️⃣ UPF에서의 Rate Limiting

#### 🔄 Token Bucket Algorithm
```
초기 상태:
├─ Token 누적 속도 = session_ambr_ul (초당 bps)
├─ 최대 버킷 크기 = session_ambr_ul / 8000 bytes
└─ 갱신 주기 = 1ms ~ 10ms

패킷 처리:
1. 경과 시간 계산
2. 토큰 재충전: tokens += (mbr_ul * elapsed_time)
3. 토큰이 패킷 크기 이상?
   ├─ YES: tokens -= packet_size; forward_packet()
   └─ NO: drop_packet(); stats++
```

#### 📊 실제 예시
```
설정: session_ambr_ul = 200 Mbps
      = 200,000,000 bps

토큰 충전 속도:
  = 200,000,000 bits/sec
  = 25,000,000 bytes/sec
  = 25 MB/sec

토큰 버킷 최대:
  = 200,000,000 / 8000
  = 25,000 bytes
  = 약 25KB (최대 버스트)
```

---

### 6️⃣ 설정 파일 예시

#### 📋 YAML 설정 (`basic_nrf_config.yaml`)
```yaml
smf:
  local_subscription_infos:
    # eMBB (Enhanced Mobile Broadband)
    - single_nssai: &embb_slice1
        sst: 1
        sd: "FFFFFF"
      dnn: "oai"
      qos_profile:
        5qi: 9
        session_ambr_ul: "200Mbps"   ← 업링크 제한
        session_ambr_dl: "400Mbps"   ← 다운링크 제한
    
    # URLLC (Ultra-Reliable Low-Latency)
    - single_nssai: &urllc_slice
        sst: 3
        sd: "000003"
      dnn: "critical"
      qos_profile:
        5qi: 80
        session_ambr_ul: "10Mbps"
        session_ambr_dl: "10Mbps"
    
    # mMTC (Massive Machine-Type Communication)
    - single_nssai: &mmtc_slice
        sst: 2
        sd: "000002"
      dnn: "iot"
      qos_profile:
        5qi: 70
        session_ambr_ul: "1Mbps"
        session_ambr_dl: "2Mbps"
```

---

### 7️⃣ 함수 호출 흐름

```
PDU Session 생성 요청 (AMF → SMF)
        │
        ↓
CreateSMContextRequest 처리
        │
        ↓
verify_sm_context_request()
        │
        ↓
get_session_ambr(snssai, dnn)  ← ⭐ 핵심 함수
        │
        ├─ find_dnn_subscription(snssai)
        │   └─ dnn_subscriptions 맵 검색
        │
        ├─ find_dnn_configuration(dnn)
        │   └─ DNN별 설정 검색
        │
        └─ qos_profile.session_ambr_ul/dl 추출
        │
        ↓
NAS 메시지 인코딩 (SessionAmbr::encode())
        │
        ↓
PFCP 메시지 생성 (session_ambr_t)
        │
        ├─ session_ambr_ul → MBR_UL
        └─ session_ambr_dl → MBR_DL
        │
        ↓
send_pfcp_session_establishment_request()
        │
        ↓
UPF로 전송 (UDP:8805)
```

---

### 8️⃣ 데이터 변환 과정

```
YAML 파일:
  "200Mbps"
        │
        ↓ 파싱 (문자열 → 정수)
        │
정수값: 200000000 bps
        │
        ├─ NAS 형식: 압축 인코딩
        │   └─ 1-3 바이트로 표현
        │
        └─ PFCP 형식: 3GPP TS 29.244
            └─ 64kbps 단위로 인코딩
                = 200000000 / 64000 = 3125
```

---

### 9️⃣ 주요 데이터 구조

#### DNN Configuration 구조
```cpp
struct dnn_configuration_t {
    std::string dnn;                    // DNN 이름
    snssai_t single_nssai;              // SNSSAI
    pdu_session_type_t pdu_session_type; // IPv4, IPv6, etc
    
    struct qos_profile_t {
        uint8_t 5qi;                    // 5QI 값
        uint8_t priority;               // 우선순위
        session_ambr_t session_ambr;    // ⭐ 세션 AMBR
        uint8_t arp_priority;
        bool arp_preempt_capability;
        bool arp_preempt_vulnerability;
    } qos_profile;
};
```

---

### 🔟 문제 해결 체크리스트

| 증상 | 확인 항목 | 명령어 |
|------|---------|--------|
| Rate limiting 안 됨 | PFCP 메시지 전송 확인 | `tcpdump -i eth0 udp port 8805` |
| AMBR 값 적용 안 됨 | YAML 구문 오류 | `yaml-lint config.yaml` |
| UPF 통계 없음 | URR 설정 확인 | `docker logs oai-upf \| grep URR` |
| DNN별 제한 다름 | DNN 이름 대소문자 확인 | `grep -i dnn config.yaml` |
| 예상 대역폭 못 도달 | Token Bucket 파라미터 | `docker exec oai-upf vppctl show counters` |

---

## 📚 참고 자료

### 주요 파일 링크
- **Markdown 분석**: `/home/inho/session_ambr_analysis.md`
- **JSON 데이터**: `/home/inho/session_ambr_analysis.json`

### 3GPP 표준
- **TS 29.244**: PFCP Protocol (session_ambr의 N4 전송)
- **TS 29.502**: SMF Service-Based Interface
- **TS 24.501**: NAS 5GSM Messages

### GitLab Repository
- **SMF**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-smf
- **UPF**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf
- **UPF-VPP**: https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp

---

## 🎯 결론

**Session AMBR은 다음과 같이 구현됨:**

1. ✅ **설정 단계**: YAML 파일에서 `session_ambr_ul/dl` 정의
2. ✅ **파싱 단계**: SMF가 `get_session_ambr()` 함수로 값 추출
3. ✅ **인코딩 단계**: NAS (UE) + PFCP (UPF) 두 가지 형식으로 변환
4. ✅ **전송 단계**: PFCP 메시지로 UPF에 MBR 값 전달
5. ✅ **적용 단계**: UPF의 QER에서 Token Bucket 알고리즘으로 Rate Limiting 수행

**Rate Limiting 메커니즘:**
- Token Bucket 또는 eBPF 기반 구현
- 초당 `session_ambr_ul/dl` 속도로 토큰 재충전
- 토큰 부족 시 패킷 드롭

**표준 준수:**
- 3GPP TS 29.244 (PFCP)
- 3GPP TS 29.502 (SMF SBI)
- 3GPP TS 24.501 (NAS 5GSM)

---

**분석 완료**: 2025-12-12  
**대상 프로젝트**: OpenAirInterface 5G Core Network v2.2.0
