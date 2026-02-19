# DC Current Demand Req/Res — 데이터 크기, 구조, MAC 계층까지의 데이터 형태

이 문서는 ISO 15118-2 및 DIN SPEC 70121에서 정의하는 **DC Current Demand Req/Res** 메시지의 **데이터 크기**, **데이터 구조**, 그리고 **MAC Layer에 도달했을 때의 데이터 형태**를 정리한 것이다. 본 레포지토리의 스키마·메시지 정의 및 전송 경로를 기준으로 작성하였다.

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

1. **ISO 15118-2:2014** (또는 최신 에디션) — Road vehicles — Vehicle to grid communication interface — Part 2: Network and application protocol requirements. CurrentDemand 메시지 정의: Section 8.4.5.4 (Current Demand), 데이터 타입 및 메시지 구조.
2. **DIN SPEC 70121** — Electric vehicle conductive charging system – Digital communication between a d.c. EV and an EVSE. DC Current Demand: Section 9.4.2.4.
3. **ISO 15118-2 EXI** — ISO 15118-2에서 참조하는 EXI(Efficient XML Interchange) 스키마 및 인코딩 규칙. 메시지의 실제 바이너리 크기 및 형태 결정.
4. **V2G Transfer Protocol (V2GTP)** — ISO 15118-2 Section 7.8 (또는 해당 절). 8-octet 헤더(Protocol Version, Inverse Protocol Version, Payload Type, Payload Length) + Payload 구조.
5. **IETF / IEEE** — TCP over IPv6, link-local 주소 사용 (ISO 15118-2 네트워크 계층 요구사항).
6. **본 프로젝트 (SwitchEV/iso15118)** — GitHub: [iso15118](https://github.com/SwitchEV/iso15118). Python 기반 ISO 15118-2 / -20 및 DIN SPEC 70121 구현, EXI 코덱, V2GTP 및 TCP 전송 로직.
