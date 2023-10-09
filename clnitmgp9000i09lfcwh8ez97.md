---
title: "git에서 특정 파일 완전 삭제"
datePublished: Mon Oct 09 2023 11:39:47 GMT+0000 (Coordinated Universal Time)
cuid: clnitmgp9000i09lfcwh8ez97
slug: git1
tags: git

---

```bash
git filter-branch --index-filter "git rm -rf --cached --ignore-unmatch 삭제할 파일 경로" HEAD

git push origin --force --all
```

---

### 출처

[Git에서 파일 완전 삭제하기 (](https://velog.io/@1ong1ong/Git%EC%97%90%EC%84%9C-%ED%8C%8C%EC%9D%BC-%EC%99%84%EC%A0%84-%EC%82%AD%EC%A0%9C%ED%95%98%EA%B8%B0)[velog.io](http://velog.io)[)](https://velog.io/@1ong1ong/Git%EC%97%90%EC%84%9C-%ED%8C%8C%EC%9D%BC-%EC%99%84%EC%A0%84-%EC%82%AD%EC%A0%9C%ED%95%98%EA%B8%B0)