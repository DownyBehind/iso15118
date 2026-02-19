# DC Current Demand Req/Res — 데이터 크기, 구조, MAC 계층까지의 데이터 형태

이 문서는 ISO 15118-2 및 DIN SPEC 70121에서 정의하는 **DC Current Demand Req/Res** 메시지의 **데이터 크기**, **데이터 구조**, 그리고 **MAC Layer에 도달했을 때의 데이터 형태**를 정리한 것이다. 본 레포지토리의 스키마·메시지 정의 및 전송 경로를 기준으로 작성하였으며, 표준 검증 시 **ISO 15118-2:2014(E)** 및 **ISO 15118-20:2022(E)** PDF를 참조하였다.

---

## 1. 개요

- **CurrentDemandReq**: DC 충전 루프에서 EV → SECC 방향. EV가 목표 전류/전압, SOC, 충전 완료 여부 등을 요청.
- **CurrentDemandRes**: SECC → EV 방향. EVSE 현재 전압/전류, 한계 도달 플래그, EVSEID 등 응답.

두 메시지 모두 **V2G_Message** 안에 **Header** + **Body** 중 하나로 담겨 EXI로 인코딩된 뒤 **V2GTP**로 감싸져 **TCP over IPv6**로 전송된다.

---

## 2. 데이터 구조 (ISO 15118-2 기준)

스키마: `iso15118/shared/schemas/iso15118_2/V2G_CI_MsgBody.xsd`, `V2G_CI_MsgDataTypes.xsd`  
구현: `iso15118/shared/messages/iso15118_2/body.py`

### 2.1 V2G 메시지 최상위 구조

| 계층 | 요소 | 설명 |
|------|------|------|
| V2G_Message | Header | SessionID(필수), Notification(선택), Signature(선택). CurrentDemand에서는 서명 없음. |
| V2G_Message | Body | 한 개의 Body 요소만 존재. CurrentDemandReq 또는 CurrentDemandRes. |

### 2.2 CurrentDemandReq (Body 요소)

| 필드명 (XSD) | XSD 타입 | 비고 | 대략 크기(참고) |
|--------------|----------|------|------------------|
| DC_EVStatus | DC_EVStatusType | EVReady, EVErrorCode, EVRESSSOC | 구조체 |
| EVTargetCurrent | PhysicalValueType | Multiplier, Unit, Value | 3요소 |
| EVMaximumVoltageLimit | PhysicalValueType | minOccurs="0" | 0 또는 3요소 |
| EVMaximumCurrentLimit | PhysicalValueType | minOccurs="0" | 0 또는 3요소 |
| EVMaximumPowerLimit | PhysicalValueType | minOccurs="0" | 0 또는 3요소 |
| BulkChargingComplete | xs:boolean | minOccurs="0" | 0 또는 1 |
| ChargingComplete | xs:boolean | 필수 | 1 |
| RemainingTimeToFullSoC | PhysicalValueType | minOccurs="0" | 0 또는 3요소 |
| RemainingTimeToBulkSoC | PhysicalValueType | minOccurs="0" | 0 또는 3요소 |
| EVTargetVoltage | PhysicalValueType | 필수 | 3요소 |

**DC_EVStatusType:**

| 필드 | XSD 타입 | 크기(참고) |
|------|----------|------------|
| EVReady | xs:boolean | 1 |
| EVErrorCode | DC_EVErrorCodeType (string enum) | 문자열 |
| EVRESSSOC | percentValueType (byte 0..100) | 1 byte |

**PhysicalValueType:**

| 필드 | XSD 타입 | 크기(참고) |
|------|----------|------------|
| Multiplier | unitMultiplierType (byte -3..3) | 1 byte |
| Unit | unitSymbolType (string enum, e.g. "A", "V", "W", "s") | 문자열 |
| Value | xs:short | 2 bytes |

### 2.3 CurrentDemandRes (Body 요소)

| 필드명 (XSD) | XSD 타입 | 비고 | 대략 크기(참고) |
|--------------|----------|------|------------------|
| ResponseCode | responseCodeType (string enum) | "OK" 등 | 문자열 |
| DC_EVSEStatus | DC_EVSEStatusType | EVSE 상태 | 구조체 |
| EVSEPresentVoltage | PhysicalValueType | 필수 | 3요소 |
| EVSEPresentCurrent | PhysicalValueType | 필수 | 3요소 |
| EVSECurrentLimitAchieved | xs:boolean | 필수 | 1 |
| EVSEVoltageLimitAchieved | xs:boolean | 필수 | 1 |
| EVSEPowerLimitAchieved | xs:boolean | 필수 | 1 |
| EVSEMaximumVoltageLimit | PhysicalValueType | minOccurs="0" | 0 또는 3요소 |
| EVSEMaximumCurrentLimit | PhysicalValueType | minOccurs="0" | 0 또는 3요소 |
| EVSEMaximumPowerLimit | PhysicalValueType | minOccurs="0" | 0 또는 3요소 |
| EVSEID | evseIDType | string, 7..37자 | 7~37 bytes (UTF-8). 표준 Table 56에는 hexBinary로 잘못 기재됨(구현은 문자열). |
| SAScheduleTupleID | SAIDType | unsignedByte 1..255 | 1 byte |
| MeterInfo | MeterInfoType | minOccurs="0" | 0 또는 구조체 |
| ReceiptRequired | xs:boolean | minOccurs="0" | 0 또는 1 |

**DC_EVSEStatusType:**

| 필드 | XSD 타입 | 비고 |
|------|----------|------|
| NotificationMaxDelay | xs:unsignedShort | 2 bytes |
| EVSENotification | EVSENotificationType (None / StopCharging / ReNegotiation) | 문자열 |
| EVSEIsolationStatus | isolationLevelType | minOccurs="0" |
| EVSEStatusCode | DC_EVSEStatusCodeType (EVSE_NotReady 등) | 문자열 |

### 2.4 DIN SPEC 70121과의 차이

- **CurrentDemandReq**: 구조는 동일. DIN은 PhysicalValue 등에 DIN 전용 타입(접미사 Din) 사용(동일 XSD 타입 기반).
- **CurrentDemandRes**: DIN에는 `EVSEID`, `SAScheduleTupleID`, `MeterInfo`, `ReceiptRequired`가 없음.  
  스키마: `iso15118/shared/schemas/din_spec/V2G_CI_MsgBody.xsd`, 구현: `iso15118/shared/messages/din_spec/body.py`

---

## 3. 데이터 크기 (논리·스키마 기준)

- **XML/논리 구조** 기준으로는 요소/속성 개수와 타입별 상한만 정의되어 있으며, 실제 전송 크기는 **EXI 인코딩 결과**에 따라 달라진다.
- **타입별 상한 요약** (XSD 기준):
  - **SessionID**: hexBinary, 최대 8 bytes (16자 hex 문자열).
  - **PhysicalValueType**: Multiplier 1 byte, Unit 문자열(짧은 enum), Value 2 bytes.
  - **percentValueType**: 1 byte (0..100).
  - **evseIDType**: 문자열 7~37자.
  - **SAIDType**: 1 byte (1..255).
  - **responseCodeType / EVSENotificationType / DC_EVSEStatusCodeType / DC_EVErrorCodeType**: 고정 열거 문자열.

### 3.1 Worst Case 크기 (MAC Layer 기준, 바이트)

Worst Case는 **MAC 계층에서 한 프레임으로 전송되는 전체 octet 수**로 정의한다. 아래에서 **사양에 명시된 것**과 **문서 작성 시 추정한 것**을 구분해 둔다.

#### 사양(ISO 15118-2 / DIN SPEC)에 있는 내용

(아래는 **ISO 15118-2:2014(E)** 본문(제공 PDF) 기준으로 확인.)

- **필드별 타입·상한**: 메시지 스키마(XSD)에 정의된 타입 제한만 있다.  
  예: `sessionIDType` hexBinary max 8, `evseIDType` string 7..37자, `meterIDType` max 32, `PhysicalValueType`(Multiplier/Unit/Value), `MeterInfoType` 내부 길이 등.  
  → 이로부터 **데이터 값 상한**(Req Body 71 B, Res Body 257 B, Header 8 B)을 **유도**할 수 있다.
- **V2GTP**: Section 7.8.3.1 (Figure 8, Table 9) — 헤더 **8 bytes**, Payload Length **0…4 294 967 295** bytes(프로토콜 상한). 메시지별 EXI 상한은 규격에 없음.
- **EXI**: Section 7.9.1(W3C EXI 1.0 사용), 7.9.1.3(EXI 옵션·프로파일). EXI payload 최대 바이트 수는 규격에 명시되어 있지 않다.

#### 사양에 없는 내용 (본 문서 추정)

- **EXI payload 예상 상한**(Req 약 150 B, Res 약 380 B): 스키마 기반 EXI 인코더를 가정한 **추정치**. 구현·옵션에 따라 달라질 수 있음.
- **MAC 프레임 전체 크기**(232 B / 462 B 등): 위 EXI 추정치에 MAC·IPv6·TCP·V2GTP 헤더를 더한 **계산 결과**이며, 사양에 정의된 값이 아님.
- **MAC(Ethernet 14 B), IPv6(40 B), TCP(20 B)** 등은 IETF/IEEE 등 일반 네트워크 사양 기준이며, ISO 15118 자체에서 “MAC 프레임 최대 길이”를 정한 것은 아님.

#### 계층별 고정/상한 값 (참고)

| 계층 | 항목 | 크기 (bytes) | 출처 |
|------|------|---------------|------|
| **MAC** | Ethernet II 헤더 | 14 | IETF/IEEE (802.1Q VLAN 시 +4) |
| **L3** | IPv6 헤더 | 40 | IETF (ISO 15118는 IPv6 사용 규정) |
| **L4** | TCP 헤더 (옵션 없음) | 20 | IETF |
| **TLS** (선택) | 레코드 헤더 + 패딩/오버헤드 | 약 21 | 일반적 추정 |
| **V2GTP** | 헤더 | 8 | **ISO 15118-2** |
| **V2GTP** | Payload (EXI) | 가변 (프로토콜 상한 2^32-1) | 메시지별 상한 미명시 → 아래는 추정 |

#### 애플리케이션 데이터 상한 (사양 스키마 기준)

**V2G_Message Header (공통):** SessionID 8 B → **8** (사양: sessionIDType max 8 octets)

**CurrentDemandReq Body:** DC_EVStatus(33) + PhysicalValueType×9(36) + boolean×2(2) → **71**  
**CurrentDemandRes Body:** ResponseCode(36) + DC_EVSEStatus(45) + PhysicalValue×5(20) + boolean×4(4) + EVSEID(37) + SAScheduleTupleID(1) + MeterInfo(114) + ReceiptRequired(1) → **257**  

위 71/257은 사양(XSD)의 타입·maxLength 등으로부터 계산한 **데이터 값 상한**이다.

#### MAC Layer 기준 Worst Case (프레임 전체, bytes) — EXI는 추정치 사용

EXI payload는 사양에 최대 길이가 없으므로, **스키마 기반 EXI를 가정한 추정 상한** Req **150 B**, Res **380 B**를 사용해 MAC 프레임 전체를 계산한 값이다.

| 구분 | CurrentDemandReq | CurrentDemandRes |
|------|------------------|------------------|
| EXI payload **추정 상한** (사양 아님) | 150 | 380 |
| V2GTP (8 + EXI) | 158 | 388 |
| TCP payload (= V2GTP) | 158 | 388 |
| TCP 세그먼트 (TCP 헤더 20 + payload) | 178 | 408 |
| IPv6 패킷 (IPv6 헤더 40 + TCP 세그먼트) | 218 | 448 |
| **MAC 프레임 (Ethernet 14 + IPv6 패킷)** | **232** | **462** |
| **TLS 사용 시** (오버헤드 약 21 B 추가) | **약 253** | **약 483** |

- **MAC 프레임** = MAC(14) + IPv6(40) + TCP(20) + V2GTP(8 + EXI). VLAN 태그(4 B) 사용 시 **+4**.
- **TLS 사용 시**: 레코드 헤더·패딩 등 약 21 B 추가로 산정.
- **DIN SPEC CurrentDemandRes**: EVSEID·SAScheduleTupleID·MeterInfo·ReceiptRequired 없음 → 462 B 미만.

### 3.2 EXI payload 상한을 다루는 외부 문서

**결론: “CurrentDemand Req/Res의 EXI payload 최대 바이트 수”를 명시한 공개·표준 문서는 없다.**

| 문서 | EXI payload 바이트 상한 여부 | 비고 |
|------|------------------------------|------|
| **ISO 15118-2:2014** | 없음 | 메시지 구조(XSD), EXI 사용 규정. 메시지별 “EXI 최대 길이” 미명시. |
| **DIN SPEC 70121** | 없음 | 동일. |
| **ISO 15118-20** | 없음 | 2세대 프로토콜; 메시지/인코딩은 다르며, EXI payload 상한 정의는 공개 검색 범위에서 확인되지 않음. |
| **W3C EXI 1.0 (Efficient XML Interchange)** | 없음 | 인코딩 포맷만 정의. “스키마·문서당 최대 인코딩 바이트” 규정 없음. 결과 크기는 스키마·옵션·내용에 따라 가변. |
| **W3C EXI Profile (limiting dynamic memory)** | 없음 | 런타임 **메모리**(문법·값 테이블 상한) 제한. **인코딩 스트림 바이트 상한**은 다루지 않음. |

따라서 **EXI payload 상한**을 얻는 방법은 다음 정도로 한정된다.

1. **사양에서 유도**: 스키마(XSD)의 필드 maxLength·타입 상한만으로 **데이터 값 상한**(예: 79 B / 265 B)은 계산 가능. EXI 바이트 수는 인코더·옵션에 따라 달라져, 동일 스키마라도 “최대 N바이트”를 사양에서 직접 주지 않음.
2. **구현 측정**: 사용하는 EXI 코덱(예: EXIficient)과 스키마로, CurrentDemand Req/Res의 **worst-case 인스턴스**를 인코딩해 측정. 이 경우 “이 구현·설정 기준 상한”만 보장됨.
3. **표준 전문 확인**: ISO 15118-2/15118-20 **전문(유료)** 의 Annex나 Implementation guideline에 “메시지별 최대 EXI 크기”가 있을 수 있으나, ISO 15118-2:2014(E) PDF(예: ISO15118-2.pdf) Section 7.8.3.1 Table 9에 Payload Length 0…4294967295 bytes. 메시지별 EXI 상한은 본문에 없음.

**대략적인 XML 크기 (참고용)**  
- CurrentDemandReq: 최소 필수만 해도 수십~100자 단위 XML, 선택 필드 모두 포함 시 수백 자 수준.
- CurrentDemandRes: EVSEID·MeterInfo 등으로 더 커질 수 있음.

**실제 전송 크기**  
- **EXI**는 스키마 정보를 이용해 요소/속성 이름을 작은 정수 등으로 대체하므로, XML보다 훨씬 작은 바이너리(수십~200 bytes 정도, 구현·옵션에 따라 다름)가 된다.
- 본 프로젝트에서는 `EXI().to_exi(to_be_exi_encoded, namespace)`로 인코딩 후 `V2GTPMessage(..., exi_payload)`로 감싸 전송한다.  
  (`iso15118/shared/states.py` create_next_message, `iso15118/shared/messages/v2gtp.py`)

---

## 4. MAC Layer까지의 데이터 형태 (프로토콜 스택)

CurrentDemand Req/Res가 **MAC 계층**에 도달했을 때는 아래 순서로 **캡슐화**된 형태가 된다.

### 4.1 인코딩·전송 경로 (본 레포 기준)

1. **애플리케이션**: V2G_Message (Header + Body에 CurrentDemandReq 또는 CurrentDemandRes)
2. **EXI**: V2G_Message를 EXI 스키마에 따라 바이너리로 인코딩 → **EXI payload (가변 길이)**
3. **V2GTP**: 8 octets 헤더 + EXI payload  
   - `iso15118/shared/messages/v2gtp.py`: `to_bytes()` → `header (8) + payload`
4. **TCP**: V2GTP 메시지 전체가 TCP payload. 포트는 SDP 협상 후 사용(일반적으로 15118).
5. **IPv6**: TCP 세그먼트를 IPv6 패킷 payload로 전송. ISO 15118는 IPv6 link-local 사용.
6. **MAC/PHY**: IPv6 패킷을 담은 이더넷(또는 HomePlug Green PHY 등) 프레임.

TLS를 사용하는 구성에서는 **TCP 위에 TLS 레코드**가 올라가고, 그 안에 다시 TCP 스트림처럼 V2GTP가 흐른다. 본 문서는 “MAC까지의 형태”만 기술하므로, TLS 여부와 관계없이 **V2GTP가 TCP payload**라는 점만 유지하면 된다.

### 4.2 V2GTP 헤더 (8 bytes, 고정)

| 오프셋 | 길이 | 필드 | 값 예 |
|--------|------|------|--------|
| 0 | 1 byte | Protocol Version | 0x01 |
| 1 | 1 byte | Inverse Protocol Version | 0xFE |
| 2–3 | 2 bytes | Payload Type | EXI encoded 등 (빅엔디안) |
| 4–7 | 4 bytes | Payload Length | EXI payload 길이(바이트). 빅엔디안. 헤더 8바이트는 제외. |

Payload Type은 `ISOV2PayloadTypes.EXI_ENCODED` (0x8001) 등으로, 프로토콜별 enum이 정의되어 있다.  
(`iso15118/shared/messages/enums.py`, `v2gtp.py`)

### 4.3 MAC 계층에서 보이는 형태 (개념)

```
+------------------+------------------+------------------+------------------+
|   MAC Header     |   IPv6 Header    |   TCP Header     |   TCP Payload    |
|   (e.g. 14 B     |   (40 B)         |   (20 B minimum) |   = V2GTP        |
|    Ethernet)     |                  |                  |                  |
+------------------+------------------+------------------+------------------+
                                                    |
                                                    v
                                    +---------------+----------------------+
                                    | V2GTP Header  | V2GTP Payload        |
                                    | 8 bytes       | = EXI(CurrentDemand  |
                                    |               |   Req or Res)        |
                                    +---------------+----------------------+
```

- **MAC payload 전체** = IPv6 패킷.
- **IPv6 payload** = TCP 세그먼트.
- **TCP payload** = **V2GTP 메시지 전체** = **V2GTP 헤더(8 bytes)** + **EXI로 인코딩된 V2G_Message(CurrentDemand Req 또는 Res 포함)**.

즉, “MAC Layer까지 갔을 때” CurrentDemand Req/Res는 **이미 EXI 바이너리**로 변환된 상태이며, **V2GTP 8바이트 헤더 뒤에 붙은 가변 길이 바이트 열**로 존재한다.  
추가로 TLS를 쓰면 이 전체가 TLS 레코드 payload 안에 들어간다.

### 4.4 정리

- **데이터 형태**: MAC에서는 **이더넷(또는 해당 MAC) 프레임 → IPv6 → TCP → (선택 TLS) → V2GTP(8B + EXI bytes)**.
- **CurrentDemand Req/Res의 “내용”**은 **EXI payload** 부분에만 들어 있으며, 사람이 읽을 수 있는 XML이 아니라 **EXI 스키마 기반 바이너리**이다.

---

## 5. 참고: 본 레포지토리 내 관련 파일

| 목적 | 경로 |
|------|------|
| CurrentDemandReq/Res 메시지 클래스 (ISO 15118-2) | `iso15118/shared/messages/iso15118_2/body.py` |
| CurrentDemandReq/Res 메시지 클래스 (DIN SPEC) | `iso15118/shared/messages/din_spec/body.py` |
| 메시지 Body 스키마 (ISO 15118-2) | `iso15118/shared/schemas/iso15118_2/V2G_CI_MsgBody.xsd` |
| 데이터 타입 스키마 (ISO 15118-2) | `iso15118/shared/schemas/iso15118_2/V2G_CI_MsgDataTypes.xsd` |
| V2GTP 메시지 형식 및 to_bytes() | `iso15118/shared/messages/v2gtp.py` |
| EXI 인코딩 및 다음 메시지 생성 | `iso15118/shared/states.py` (create_next_message), `iso15118/shared/exi_codec.py` |
| TCP 전송 | `iso15118/shared/comm_session.py` (send, V2GTPMessage) |
| CurrentDemand 타임아웃 (ISO 15118-2) | `iso15118/shared/messages/iso15118_2/timeouts.py` (CURRENT_DEMAND_REQ = 0.25 s) |
| CurrentDemand 타임아웃 (DIN SPEC) | `iso15118/shared/messages/din_spec/timeouts.py` (CURRENT_DEMAND_REQ = 0.25 s) |

---

## 6. References

1. **ISO 15118-2:2014(E)** — *Road vehicles — Vehicle-to-Grid Communication Interface — Part 2: Network and application protocol requirements.* 본 문서의 데이터 구조·크기·프로토콜 스택 검증은 이 표준 PDF를 참조함.  
   - CurrentDemand 메시지: Clause 8.4.5.4 (Current Demand).  
   - V2GTP: Section 7.8.3.1 (Figure 8, Table 9) — 헤더 8 octets, Payload Length 0…4 294 967 295 bytes.  
   - EXI: Section 7.9.1 (W3C EXI 1.0 사용), 7.9.1.3 (EXI 옵션·프로파일).  
   - 스키마: Annex C (Schema definition).  
   - 본 문서 검증 시 참조한 파일: **ISO15118-2.pdf** (예: `ISO15118-2.pdf` 또는 동일 표준 PDF).
2. **ISO 15118-20:2022(E)** — *Road vehicles — Vehicle to grid communication interface — Part 20: 2nd generation network layer and application layer requirements.* 2세대 프로토콜. V2GTP(7.8), EXI·프레젠테이션 계층(7.9), TCP/IPv6(7.6, 7.7) 등 스택 요구사항 정의. 메시지 세트는 ISO 15118-2와 다르며, 본 문서의 CurrentDemand(1세대)와 동일한 메시지명/구조는 아님. 참조 파일: **iso15118-20.pdf**.
3. **DIN SPEC 70121** — Electric vehicle conductive charging system – Digital communication between a d.c. EV and an EVSE. DC Current Demand: Section 9.4.2.4.
4. **W3C EXI 1.0** — Efficient XML Interchange (EXI) Format 1.0, W3C Recommendation (March 2011). ISO 15118-2 Normative references에서 참조.
5. **IETF / IEEE** — TCP over IPv6, link-local 주소 사용 (ISO 15118-2 Section 7.6, 7.7).
6. **본 프로젝트 (SwitchEV/iso15118)** — GitHub: [iso15118](https://github.com/SwitchEV/iso15118). Python 기반 ISO 15118-2 / -20 및 DIN SPEC 70121 구현, EXI 코덱, V2GTP 및 TCP 전송 로직.
