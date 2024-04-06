<details>

<summary><strong>htARTE (HackTricks AWS Red Team Expert)</strong>를 통해 AWS 해킹을 처음부터 전문가까지 배워보세요<strong>!</strong></summary>

HackTricks를 지원하는 다른 방법:

* HackTricks에서 **회사 광고를 보거나 PDF로 HackTricks를 다운로드**하려면 [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)를 확인하세요!
* [**공식 PEASS & HackTricks 상품**](https://peass.creator-spring.com)을 구매하세요.
* 독점적인 [**NFT**](https://opensea.io/collection/the-peass-family) 컬렉션인 [**The PEASS Family**](https://opensea.io/collection/the-peass-family)를 발견하세요.
* 💬 [**Discord 그룹**](https://discord.gg/hRep4RUj7f) 또는 [**텔레그램 그룹**](https://t.me/peass)에 **참여**하거나 **Twitter** 🐦 [**@hacktricks_live**](https://twitter.com/hacktricks_live)를 **팔로우**하세요.
* **HackTricks**와 **HackTricks Cloud** github 저장소에 PR을 제출하여 여러분의 해킹 기법을 공유하세요.

</details>


# CBC

**쿠키**가 **사용자 이름**만 있는 경우 (또는 쿠키의 첫 번째 부분이 사용자 이름인 경우) "**admin**" 사용자 이름을 가장하고 싶다면, **"bdmin"** 사용자 이름을 생성하고 쿠키의 **첫 번째 바이트**를 무차별 대입(bruteforce)할 수 있습니다.

# CBC-MAC

**Cipher block chaining message authentication code** (**CBC-MAC**)는 암호학에서 사용되는 방법입니다. 이 방법은 메시지를 블록 단위로 암호화하고, 각 블록의 암호화가 이전 블록과 연결되도록 작동합니다. 이 과정은 **블록 체인**을 생성하여 원본 메시지의 단 하나의 비트라도 변경하면 암호화된 데이터의 마지막 블록에 예측할 수 없는 변경이 발생하도록 합니다. 이러한 변경을 수행하거나 되돌리기 위해서는 암호화 키가 필요하여 보안이 보장됩니다.

메시지 m의 CBC-MAC을 계산하기 위해, m을 초기화 벡터를 0으로 설정하여 CBC 모드로 암호화하고 마지막 블록을 유지합니다. 다음 그림은 비밀 키 k와 블록 암호 E를 사용하여 메시지의 CBC-MAC을 계산하는 과정을 스케치한 것입니다:

![https://wikimedia.org/api/rest\_v1/media/math/render/svg/bbafe7330a5e40a04f01cc776c9d94fe914b17f5](https://wikimedia.org/api/rest\_v1/media/math/render/svg/bbafe7330a5e40a04f01cc776c9d94fe914b17f5)를 사용하여 블록으로 구성된 메시지의 CBC-MAC 계산

![https://upload.wikimedia.org/wikipedia/commons/thumb/b/bf/CBC-MAC\_structure\_\(en\).svg/570px-CBC-MAC\_structure\_\(en\).svg.png](https://upload.wikimedia.org/wikipedia/commons/thumb/b/bf/CBC-MAC\_structure\_\(en\).svg/570px-CBC-MAC\_structure\_\(en\).svg.png)

# 취약점

CBC-MAC에서 일반적으로 사용되는 **IV는 0**입니다.\
이는 알려진 두 개의 메시지 (`m1` 및 `m2`)가 독립적으로 2개의 서명 (`s1` 및 `s2`)을 생성한다는 문제가 있습니다. 따라서:

* `E(m1 XOR 0) = s1`
* `E(m2 XOR 0) = s2`

그런 다음 m1과 m2를 연결한 메시지 (m3)는 2개의 서명 (s31 및 s32)을 생성합니다:

* `E(m1 XOR 0) = s31 = s1`
* `E(m2 XOR s1) = s32`

**암호화의 키를 알지 못해도 계산할 수 있습니다.**

8바이트 블록으로 이름 **Administrator**를 암호화하는 경우를 상상해보세요:

* `Administ`
* `rator\00\00\00`

**Administ** (m1)라는 사용자 이름을 생성하고 서명 (s1)을 검색할 수 있습니다.\
그런 다음, `rator\00\00\00 XOR s1`의 결과로 된 사용자 이름을 생성할 수 있습니다. 이렇게 하면 `E(m2 XOR s1 XOR 0)`를 생성하게 되고 이는 s32입니다.\
이제 s32를 전체 이름 **Administrator**의 서명으로 사용할 수 있습니다.

### 요약

1. 사용자 이름 **Administ** (m1)의 서명인 s1을 얻으세요.
2. 사용자 이름 **rator\x00\x00\x00 XOR s1 XOR 0**의 서명인 s32를 얻으세요.
3. 쿠키를 s32로 설정하면 사용자 **Administrator**의 유효한 쿠키가 됩니다.

# IV 제어를 통한 공격

IV를 제어할 수 있다면 공격은 매우 쉬워질 수 있습니다.\
쿠키가 단순히 암호화된 사용자 이름인 경우, 사용자 "**administrator**"를 가장하려면 사용자 "**Administrator**"를 생성하고 해당 사용자의 쿠키를 얻을 수 있습니다.\
이제 IV를 제어할 수 있다면, IV의 첫 번째 바이트를 변경하여 **IV\[0] XOR "A" == IV'\[0] XOR "a"**로 설정하고 사용자 **Administrator**의 쿠키를 다시 생성할 수 있습니다. 이 쿠키는 초기 **IV**로 **administrator** 사용자를 **가장할 수 있는** 유효한 쿠키가 될 것입니다.

## 참고 자료

자세한 내용은 [https://en.wikipedia.org/wiki/CBC-MAC](https://en.wikipedia.org/wiki/CBC-MAC)에서 확인할 수 있습니다.


<details>

<summary><strong>htARTE (HackTricks AWS Red Team Expert)</strong>를 통해 AWS 해킹을 처음부터 전문가까지 배워보세요<strong>!</strong></summary>

HackTricks를 지원하는 다른 방법:

* HackTricks에서 **회사 광고를 보거나 PDF로 HackTricks를 다운로드**하려면 [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)를 확인하세요!
* [**공식 PEASS & HackTricks 상품**](https://peass.creator-spring.com)을 구매하세요.
* 독점적인 [**NFT**](https://opensea.io/collection/the-peass-family) 컬렉션인 [**The PEASS Family**](https://opensea.io/collection/the-peass-family)를 발견하세요.
* 💬 [**Discord 그룹**](https://discord.gg/hRep4RUj7f) 또는 [**텔레그램 그룹**](https://t.me/peass)에 **참여**하거나 **Twitter** 🐦 [**@hacktricks_live**](https://twitter.com/hacktricks_live)를 **팔로우**하세요.
* **HackTricks**와 **HackTricks Cloud** github 저장소에 PR을 제출하여 여러분의 해킹 기법을 공유하세요.

</details>