# PLURA-EDR Agent Status via Windows WMI

PLURA-EDR (Agent)는 보안 상태 정보를 Windows WMI에 등록하여 PDP가 동일한 PC 내에서 안전하게 조회할 수 있도록 합니다.  
이를 위해 커스텀 네임스페이스(root\PLURAEDR)와 클래스(AgentStatus)를 생성하고, 악성코드 탐지 여부, 무결성 검증 결과 등의 상태를 실시간으로 업데이트합니다.

---

## ✅ 구조 요약

| 역할              | 설명                                             |
| --------------- | ---------------------------------------------- |
| **PLURA Agent** | Windows WMI의 커스텀 네임스페이스와 클래스를 생성하고, 여기에 상태를 등록 |
| **PDP Agent**   | WMI API 또는 PowerShell을 통해 로컬 상태 정보 조회          |
| **통신 방식**       | 로컬 WMI / CIM 쿼리 (네트워크 불필요)                     |
| **보안**          | 시스템 내부 전용, 외부 포트 개방 없음, 커널 기반 접근 통제            |

---

## ✅ A. PLURA Agent 측 – WMI 상태 등록

### 1. WMI 네임스페이스 및 클래스 생성 (C#, PowerShell, C++ 등 가능)

예: 네임스페이스 `root\PLURAEDR`, 클래스 `AgentStatus`

```mof
// AgentStatus.mof
#pragma namespace("\\\\.\\root\\PLURAEDR")

class AgentStatus
{
  [key] string AgentId;
  string Hostname;
  boolean MalwareDetected;
  boolean RealTimeProtection;
  boolean IntegrityPassed;
  datetime LastScanTime;
};
```

### 2. MOF 등록 (설치 시 한 번만 수행)

```powershell
mofcomp AgentStatus.mof
```

### 3. C# 또는 PowerShell로 WMI 객체 쓰기

```powershell
$instance = ([WMIClass]"root\PLURAEDR:AgentStatus").CreateInstance()
$instance.AgentId = "win10-abcd-1234"
$instance.Hostname = $env:COMPUTERNAME
$instance.MalwareDetected = $false
$instance.RealTimeProtection = $true
$instance.IntegrityPassed = $true
$instance.LastScanTime = (Get-Date).ToUniversalTime()
$instance.Put()
```

* 이 코드는 주기적으로 실행하거나, 상태 변경 시 수행합니다.

---

## ✅ B. PDP Agent 측 – WMI 상태 조회

### 1. PowerShell 예제

```powershell
$agentStatus = Get-WmiObject -Namespace "root\PLURAEDR" -Class "AgentStatus" | Where-Object { $_.AgentId -eq "win10-abcd-1234" }

if ($agentStatus.MalwareDetected -eq $true) {
    Write-Output "악성코드 탐지됨 → 접근 차단"
} else {
    Write-Output "정상 상태 → 접근 허용"
}
```

### 2. C# 예제

```csharp
var searcher = new ManagementObjectSearcher("root\\PLURAEDR", "SELECT * FROM AgentStatus WHERE AgentId='win10-abcd-1234'");
foreach (ManagementObject queryObj in searcher.Get())
{
    Console.WriteLine("MalwareDetected: " + queryObj["MalwareDetected"]);
}
```

---

## ✅ 보안 및 안정성 포인트

| 항목            | 이유                                               |
| ------------- | ------------------------------------------------ |
| ❌ 외부 노출 없음    | WMI는 로컬 시스템 내 커널 레벨 IPC로만 동작                     |
| ✅ 메모리 기반      | 상태가 메모리에 상주하며 빠르게 조회 가능                          |
| ✅ 관리자 권한 필요   | 일반 유저가 생성하거나 위조하기 어려움                            |
| ✅ 안정성 높음      | WMI는 Windows 자체 구성 요소로 Buffer Overflow 취약점 거의 없음 |
| ✅ 로그 없이 통신 가능 | HTTP처럼 네트워크 흔적 없음, 탐지 회피 목적에서도 강함                |

---

## ✅ 보완 제안 (선택)

| 항목            | 설명                                      |
| ------------- | --------------------------------------- |
| TTL/유효기간      | PDP는 `LastScanTime`이 너무 오래된 경우 무효 처리 가능 |
| Agent 제거 시 정리 | Agent uninstall 시 `Remove-WmiObject` 처리 |
| 사용자 정의 권한 설정  | WMI 클래스 접근 권한 제한 가능 (WBEM ACL)          |

---

## ✅ 결론

| 항목     | 선택 이유                                    |
| ------ | ---------------------------------------- |
| 성능     | 매우 빠름 (메모리 접근 수준)                        |
| 보안     | 외부 포트 미사용, 위조 어려움, OS 커널 보호              |
| 관리 편의성 | Windows 서비스로 실행 시 지속적 갱신 가능              |
| 적합성    | Buffer overflow 걱정 없이 내부 통신만 사용하는 환경에 최적 |

---

다음으로 확장 가능:

* MOF 파일 템플릿 (커스텀 필드 포함)
* C#용 AgentStatus 등록/삭제 라이브러리
* PowerShell 자동 설치 스크립트
