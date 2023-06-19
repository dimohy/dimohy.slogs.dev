---
title: "힙 할당을 감지하는 방법 | Bart Wullems"
datePublished: Mon Jun 19 2023 05:51:48 GMT+0000 (Coordinated Universal Time)
cuid: clj2fvj9e001109l35vve7wgl
slug: bart-wullems
canonical: https://bartwullems.blogspot.com/2023/06/how-to-detect-heap-allocations.html
tags: performance, net, dotnet

---

> Bart Wullems님의 [How to detect heap allocations](https://bartwullems.blogspot.com/2023/06/how-to-detect-heap-allocations.html)을 번역하였습니다.

---

**몇 주 전에 정적 익명 함수와 람다를 사용할 때 힙 할당 횟수를 제한하는 데 도움이 되는 방법**에 대해 이야기한 적이 있습니다. 이 포스팅 이후 한 동료로부터 이러한 할당을 감지하는 방법에 대한 질문이 있었습니다.

좋은 질문입니다! 몇 가지 방법을 알려드리겠습니다.

먼저 일반적인 답변을 드리고 두 가지 구체적인 도구에 대해 자세히 살펴보겠습니다.

람다를 사용할 때 과도한 할당을 발견하려면 메모리 프로파일러 도구를 사용하여 `*_DisplayClass*` 또는 다양한 `Action*` 및 `Func*`의 변종 할당을 찾을 수 있습니다.

이 정보를 통해 무엇을 찾아야 하는지 이미 알고 계실 것입니다.

## Visual Studio 성능 프로파일러

가장 먼저 도움이 되는 옵션은 Visual Studio의 성능 프로파일러입니다.

* 빌드 유형을 **릴리스**로 변경합니다.
    
* 이제 **디버그** -&gt; **성능 프로파일러...**로 이동합니다. 여러 프로파일링 대상을 사용할 수 있지만 **.NET 개체 할당 추적** 옵션을 사용하고자 하므로 이 확인란을 선택합니다.
    

* **시작** 버튼을 클릭하여 도구를 실행합니다.
    

![](https://lh3.googleusercontent.com/DFYldZQ7eDK3VedQlbLqJPIY9oJpCJaQnnBPdbac0_2ZxEbkh2ArS6XKWFIQ1scQd2s68H0hnF89ndnmIdDOmIP20ITWXNHaOh5gzXLEHb8jEvYbkx4ickDnu9U3SqGDimyhlueUv1A8u7YBlqgMMxUpak6oP1HJ_1Pnbvu7KegSQvWzIoK9hAcS_5vJx-P9bqfxhETruq4ROeRGwpheO3hq65QVw_RH1ZpXjIRGOb043QC_6lGcl_UhkskCDlf1ONbB9DqnASKlZy0JcspioOaRYWXNn5oCbEgEJD0SLUL0xvfn81ixbS7QtEcOlv_3VvQKp57-ZQUMmCAG3CeFQmg2lDTcOZlK3IQBxH9R1jmwYHoZbAgEhOcaoe7KZ494pSQhi6lGNU1Pe1WgFIN3_x7R7wmME5NPF2Iek4A2Timm7o4oM88VBfFtYiQq2bJgepC3OUQTqaB4PXkN1cBkpcQV68UJZ0c-WD5Esjh7woS-wECHgFrg61Q2b0VH3A_iwe2GpzXgtQA_k0jjsHpKMp1e4V0FmBQqeW6L-ew9Fqwc4XV6FLiRvSMq5AbT1MiBwqT92efKjS21sed15pOr8HGMCujxWAc_WoHQToo0Dali8zAGVdb7YPpfy9NyvehKrIAzIoLtkEKr5Rlr9C9cbuhht6vY_V1BPZy2bmw4VgJO9FR9c62E4ub9YqgvBLee3o7O808NA6331ShP6vYEam7znXdZSIgCrci07t9ygBrfPy5vpnC9RPGzkhJVL6kolkwRgUt9D6_j0AwYuPwJOxT_jToWEogNCzbPu-MQKTqBBRyjDcRlcXkE_J1HYC3p-1fCfJZ3FDTiqR5Qy13o96MY-QccftTbAiGods73jGaKpRDhsDBs0ysnDssEukVFEODo8A9bcBU56GeN8E-WP7kjY7mfYd8tWmkRQq0-V1sS=w2497-h1067-s-no?authuser=0 align="left")

* **시작** 버튼을 클릭하여 도구를 실행합니다.
    

![](https://lh3.googleusercontent.com/lXrFevS3ph1rXfSx6n0wXKOIh80xl48UaFIy7Y_bmOSYt_TOSH_3Ti-W5UBlEP--x2Z-xhqUcKEgsPg0TVINenuYdvgOERFVCHOdKsCwrL5jHeSAkZOh74TgT3Y4Yy6c9bKuo3MCiizcK8P4T3eq768F0zu4eXKR6VoGv32a6jvsTkOISiws9IYiN71W0oqvlW3OSx3jCgBgDQiFsJaiQEURmM7TvtI6s5OnC7-0pBhFdRtDaUao0ofgy3jxQhmfDxrccx8f1Ch-dtOsLzaBFY75a0clIToo9O7QxsUUpd9qXk0tSo9GUpTgL_mK-w_vOr9-V-NrQvi8bj25BFDv_6MXEb8ta0XUzTv8MGSKxqMVvbOUjbyF1ErnPpDmWSE9czn8ssJGG97a1vusW8AzhTjSfjOL2VUnlHO7XbdglQJ8kqUekVYha7yGMxUDww-hcgpko4SZnKWoYCBJbkBu6dZXl8LU7VoFe6FJwtWcWfBhFI5GvAZd-XsRBZk0sQ4FumOZpHpNjL-6kQh6RHZySt9xj7n1FwLbM_lfmG_1hBXFjTi7Br7ip7dT1tVRfQYb0-6sbYQtx53_bwZbCI1ivPZLMlvk_hIqKxUZro2tdfdjChBivG_o8Q6I2O4J3G8UfIlk1Jm5vBWUUzN1eflDiEgbI3zGpJagSY016KdmJe7C8ArDm_Y84qLu06KvbHn-wK2VAB5ybnvSAybt2li20voRNdDCDq9hjkyqpYSwhxdJ1cmecV_kQA5B6Z9rhkjMPGMeQBiqhBWOXs3-lOpEnuimGOqxANNwo_qzlyG2gDFY8klsMbHl3Aw_gkM9RxS_T1HDrrOCoA78zSTY3QEH08W1Ii6zqCTwiVw5nio14PAV4VkUpPeQtM2OywVAEIvbmWZCsiAu3vv52MSK7sxgcxoNPxNeZw4vZVR7doC8o6BT=w2618-h1316-s-no?authuser=0 align="left")

* 프로파일링된 애플리케이션을 닫거나 **수집 중지**를 클릭하면 **할당** 탭에서 모든 할당을 볼 수 있습니다.
    

![](https://lh3.googleusercontent.com/y7cMgRQMd4IhGh6-c2EUVfyBOWMT2nOeMfHvxs8w2vMez-taZMYA3ot2FBbmVytmWLLLTfw8NyyCXvdcGwucOgN3lDxQKVCUbHKZyKrYTt-b9qyEc3Eiujdom61Qhtyg_HVzpkss07VdKisNsisoo1W90ywrf77fxIYw6yNCuybh5mcch5_g1To0sRelL-EFb_KrWUzdk2cTYnKEb-fAekAjV2uxO3YqVrP8_FPHY8JirgMwAacBbG8qZUHE9AyH0qC-PzZIr-GPNJEUwtqCHs710aB2GbruqXv_xotfCJHEv2FY5wm9sAWEmRNC5kQ8JzxTH7305Wk3SNgI3sAXTPTAIGe6JF7XqoporrPeOUOmBgUHifmFRiK1gB2zltySU5GWJBQ2fmoveo4QqwJXV920oI9rjRqhEDdQvucsdhkc2Hr2iYX-86rcVdg0Bpgi1tss19BZNvuLfjuI-_RQHDz30s3ee5Yhsb873x1bVxIxDi9WDxrMV0MWjdCQKmtnj5ZWp71LnflwG6ZroDYboBmtst7tBuHLrW1xplVffHlZj01QxjK8tWbQAvv4qtw8D5kCe79J9tRTLXsKzp7IWCCbNomtOOqLt9A8ELNF8jAQKewp-VTGJE-UbEiVQt3lwFfciJv5f-83Mbulra1JJkqS5sPv7pFb197UMoIZCPHufg6CbmO85LQP8vOz4WtOc0Fkze9u9e3-4u2n_kMpl1EW2rYjEhLIm4rZL0Lppq_Mbybb_ig8bzchjc7HMssD0kVEB9VHw4fVDIseXhRWiD2u1pdPuGxTgv3qpb3PQCz0lv5WqOBxfS8bPA_J0GGh_su30fjpRrenrMTlnSrwdWKD7qsiThe1CtGKV_tde2Rzkz5brjRR0C7cM2hamcvORWAxuMvDr3e3ATioU2fgZ8DKtDOaYGX83qCvL0jLEjwl=w1212-h506-s-no?authuser=0 align="left")

자세한 정보 [.NET 개체의 메모리 사용량 분석 - Visual Studio(Windows) | Microsoft Learn](https://learn.microsoft.com/en-us/visualstudio/profiling/dotnet-alloc-tool?view=vs-2022)

## Roslyn Clr 힙 할당 분석기

또 다른 옵션은 명시적 할당과 박싱, 디스플레이 클래스(클로저라고도 함), 암시적 델리게이트 생성 등과 같은 많은 암시적 할당을 감지할 수 있는 Roslyn 기반 C# 힙 할당 진단 분석기입니다.

프로젝트에 NuGet 패키지로 직접 설치할 수 있습니다:

```bash
dotnet add package ClrHeapAllocationAnalyzer
```

다른 분석기와 마찬가지로 인라인 힌트를 제공합니다:

![](https://lh3.googleusercontent.com/kondsTh-Vix_mH88M0GQPoBeE8uCSGqrCnM_7CCIF8CJavNtyiq-JNm2eZ-rZo7XcqPSzbbqcAUh9M2Cy0Pa1nz7aNObhHmwBM0HidwkEEKRZZsZKmyTehu8DogZMo4HN-EjKyvxBFzRYglvvudk3ejuUjzuZ9TXmqUPnWZiZyo8QTACVH3oBhAxRaB01vEZVoZPd20_3Nc8Yhmej6g_Lb_h3hfZMpxv7JOPs_DY8G2nk1xgeGdN7MLgV36s0FScuH-cjfCWdp0VlJhY6Mmx8Tf3waUPK0K7Db5lWR6VgaB_RkoZ2f5_xeafVbOiFs2wj2QGFO9YmRoEeUzTebSX0dt6ey5ttX3Bo0t8itFHoR6srFDW4RA7hoEfMGunEC2hSP4QhMk5nPTRBz4eyGeD6AauYD9HMo8MQDpPe-xkUY3Xm_BcR31E_8uPEHsSE0KmPLcwKGemeRpTOzS645INn0Sgi3GBn7CfS61mVDazFhRHIbCr2edol32l1DQZnspyzhehmmRlcDoaMjrYC3SD4wBiBcEhlnJuzkW542LbrbV72RvGIeg0pISDBFWo73qEk_POJxrto25zacNHYcNyk_r92dkEwj8LGH3qOwIXNZ8OCyOc6Hvu2M26ZwLTMYFlyVTZRnB4JAmw2SUOhehCQvFedjTF6m-KACJnLzOicChQh8o2BMnCNoIjJkrwzrqaO9WsC41rEtWbP7ooi5JPuOsZYWdBSNqGXuVLa845QD4KV8yPiIep7cMBa3j-XMnz09quG2zZwSeLP8nlO2Bk0_8glupH3iEk0Kz7TMps1MnSrnhyW7UeMCquuspH8cWEQPFh0sJpw4CeQ-qThWEHODE84opd3uSg8s8P95w25zwR58-NYC8rCVaM43VNTp9OSFczyr3VEJUd3u8Px6D_osSvcIFjK6OI7tQudooORr6p=w1446-h595-s-no?authuser=0 align="left")

관련 경고 확인:

![](https://lh3.googleusercontent.com/3RuZzq0OLXYMwAkF33wWThBofApQrWHj9AANb5i5yQvuL275waP281Xytqrqnt-pJOgB418ixHR7o-jsQF2IOR7jHGxeIa_aYyXBMJNlEEHTgOtwGR-i-nA7i1nqBPdW2Z1hyln7jc7wZn3G3nhcp0uElTqsebkdU7XHBoDwacmFIuU1wWDlj2FvFECpxGqKxttTtcdj2Y9Q-cBV6y3kaMx7LT4WtSo0QG4UtHTWnJk2V3P0TuPEIur27Eng7cnyTZUTerFRgR6mk94W5wBg6H001skDRQLbb6OHfzReUX3QZ7payWz7E4yDV2eci6Vy7aJH4_oXUcKg60Vp4uieRuX8SCq6MLmBtJnKpcFWEx8wUDQxIFkGcejfPHo0JFYiZIDmqZMbM-8lYrX02a4wQTy8O_Qb16fAAntuuNkc4eqJCqNFCTdbwPcLSEG94QznPrVaO3PWC8KYz7_0JSgtTzhiCbYFehBF64NMLuNLN_i-1UjY8lnQnf_XhWAinitoeYLIhJpu2UfiAELzd93gGAQhToFv-iWS36CbenRxM_K8t3qErRHcXHDv8GTOcrFwrq0GT8PCmJSBqZILTakL6Z7m0q-z88cPzoDBYabWR2A9fGEGLHnhxIkHwJrkEuC6TxF5WKcapRNeSJJA8O7igLj_lpd6WIae0YBkU6NFaGLyMbMw07cqQbEzOAXsYxkL763KpG0kVXlVSI_LPbs-QRzleMd65nDQ5qA680tEV3TCMcM87nVYttbW8CqRhQ4Auaab8eIsAH5740wxr9QJBpUlHZloSkOAYtJkDxZcEre8LvQ4mFwk8zAlQTa0rl__uNMtq-5iq1lB2XoMafmuKu-sw4MVBR-M4CFFDKAej12dONxy1nFfZ7lCpdK7dMJ_RYVhk5PReGXWPafW8M60UjXy4glOIySHyqeHBm1htJnQ=w1377-h293-s-no?authuser=0 align="left")

실제로 작동하는 모습을 보고 싶으시다면 다음 동영상을 참조하세요:

%[https://youtu.be/Tw-wgT-cXYU] 

**참고**: JetBrains Rider를 사용하는 경우, [힙 할당 뷰어](https://plugins.jetbrains.com/plugin/9223-heap-allocations-viewer) 플러그인을 사용하여 동일한 작업을 수행할 수 있습니다.

---

%[https://bartwullems.blogspot.com/2023/06/how-to-detect-heap-allocations.html]