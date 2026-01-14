# 기본 개념
> [!info] Hypervisor
> 하나의 물리적 컴퓨터 위에 여러 개의 가상 머신(`VM`)을 실행할 수 있도록 하는 소프트웨어를 말한다
> > [!faq] VM?
> > Virtual Machine의 약자로 하나의 운영체제를 독립적으로 실행할 수 있는 가상 컴퓨터를 말한다
> ```mermaid
> block-beta
> columns 1
> 	block:VM
> 		columns 2
> 		block:VM1
> 			columns 2
> 				APP1
> 				APP2
> 		end
> 		block:VM2
> 			APP3
> 			APP4
> 		end
> 		3["Bin/Libs"]
> 		4["Bin/Libs"]
> 		1["Guest OS"]
> 		2["Guest OS"]
> 	end
> 	Hypervisor
> 	5["Host OS"]
> 	6["Physical Server"]
> ```
> 이렇게 `VM`위에 하나의 `OS`를 설치하고 그 위에 서비스를 설치하여 실행할 수 있다
> `VM` 위에 하나의 `OS`가 독립적으로 돌아가고 있기에 서비스 하나하나가 무겁다
> 그래서 부팅 시간도 느리다

> [!info] Container
> `App`과 필요한 라이브러리, 의존성, 설정 파일 등을 하나의 독립된 단위로 묶어서 운영체제의 커널을 공유하지만 독립된 환경에서 실행하는 기술이다
> 가장 이 기술을 먼저 도입한 서비스가 `Docker`이다
> ```mermaid
> block-beta
> columns 1
> block:Docker
> 	columns 4
> 	App1 App2 App3 App4
> 	Bin/Lib1 Bin/Lib2 Bin/Lib3 Bin/Lib4
> end
> 1["Docker Engine"]
> 2["Host OS"]
> 3["Physical Server"]
> ```
> `Docker`라는 베이스 위에 소스코드 이미지가 실행되어 각각 독립된 상태로 돌아간다
> 그래서 `container` 하나하나가 가볍다

자세한 내용을 좀 더 보려면 ([[20260113]]) 문서를 참고하시길

---
# 둘을 보안적 관점으로 비교를 한다면?
##### 1. Hipervisor는 
하드웨어부터 서로 격리되는 구조이기에 서로 다른 컴퓨터에서 돌아간다고 봐야함
즉, 애초에 다른 컴퓨터로 서비스가 격리되기에 하나의 서비스가 취약해서 다운이 되더라도 나머지 서비스에 보안적 영향이 없음
-> 공격자가 Hipervisor를 뚫지 않는 한 다른 VM들은 안전함

##### 2. Container는
OS 단계에서 가상화를 진행함
즉, 같은 `kernel`을 공유하고, Namespace, cgroups 같은 커널 기능에 의존하여 격리하는 구조임
-> 커널의 취약점이 곧 모든 컨테이너의 취약점임

이를 더 자세하게 표로 비교하면

| 항목 | Hypervisor | Container |
| :--: | :--: | :--: |
| 커널 수 | VM마다 개별 커널 | 단일 커널 공유 |
| 침해 범위 | VM 내부로 제한 | 호스트 전체 확산 가능 |
| 탈출 공격 | 매우 어렵고 희귀 | 비교적 빈번 |
| Zero-day 영향 | VM 단위 | 전체 컨테이너 |

-> Container가 공격 표면이 더 넓고, Hipervisor가 공격 성공 난이도 가 더 높음

## Escape Attack 가능성 비교
##### 1. VM Escape
- Hipervisor 취약점 필요
- 발견 빈도와 악용 사례가 낮음

##### 2. Container Escape
- 커널 취약점 필요
- 권한이 잘못 설정되어 있을 경우 (`--privileged`)
- Docker socket이 노출되어 있는 경우
- seccomp / Apparmor 가 적용되어있지 않은 경우
-> 이런 경우들에 일어날 수 있음

# Docker가 CVE 취약점 95% 감소시킨다던데?
##### 1. 오해
위 내용을 포함해서 사람들은 `Container`는 `kernel`을 공유하니 무조건 위험하다 라는 생각을 가질 수 있는데, 이는 오해이다.
원래는 `kernel`이 뚫리면 `Container`는 자연스럽게 취약한 것이 정상이지만, 이를 방어하는 `Docker`의 기술이 존재한다
바로 `Hardened Images`와 `SBOM`이다

##### 2. Hardened Images? SBOM?
- 먼저 짚고 넘어갈 사항은 `CVE` 취약점의 80~90% 정도는 `APP`이 아닌 불필요한 `package`에서 발생한다는 것이다
- 즉, 과잉 포함(`over-inclusion`)이 취약점의 주요 원인인 셈이다

1. Hardened Images
`Hardened Images`의 핵심 철학은 "없으면, 취약할 수 없다" 이다
그리하여, 최소한의 구성만을 가지고 있게끔 한다.
	- `shell, package manager, compiler, debuging tool` 모두 없다
-> 즉, `CVE`가 자주 발생하는 `package` 자체가 이미지에 존재하지 않는 것이다


2. SBOM
`SBOM`은 `Software Bill of Materials`의 약자로 취약점이 없는 이유를 증명하는 도구이다
이 안에는 라이브러리, 버전, 의존성, 해시값 등이 나열되어 있다
이를 통해 `CVE` 매칭 정확도가 상승했으며, `False positive` 또한 제거되었고, `SolarWinds`, `Log4Shell`과 같은 공급망 공격 이슈에서 영향 범위를 즉시 확인하고 대응할 수 있게 되었다

3. 결과적으로
범용 `OS`를 활용하고 `package`가 과잉되었던 문제를 `Hardened Images + SBOM`을 통해 최소한의 구성과 공격 표면을 제거하고 구성 요소를 완전 가시화 함으로서 문제를 해결함
-> 한마디로 말하면 `CVE`를 고쳐서 제거한 것이 아닌 `CVE`가 생길 구조를 **없애서** 해결한 것이다

# 결론
하드웨어적 보안을 추구한다면 `Hypervisor`를 사용하면 된다.
하지만 `Component` 방식이 빠르고 편리하다
기본적으로 `Component` 방식이 `Hypervisor` 방식에 비해 보안이 취약한 것은 사실이지만 이를 보완하기 위한 `Docker`의 `Hardened Images + SBOM` 기술이 존재한다
