---
title: "[context] arithmetization"
tags:
  - zkp
---

Arithmetization (算術化) 是將一般陳述或問題轉化為一組待驗證或求解的方程式的過程。驗證這些陳述或是問題時，只需將這些值代入方程式並 evaluate 方程式的左右兩側是否成立。一個「好的」算術化是產生出一個數學表達式，可以以最少的計算去 evaluate 整個方程式。

一個常見的 Arithmetization 是 R1CS (rank 1 constraint system)，被 Pinocchio, Groth16, Bulletproofs 採用。而 Plonk 系列的 Arithmetization 則稱之為 PLONKish Arithmetization。

## reference

- https://github.com/sec-bit/learning-zkp/blob/master/plonk-intro-cn/1-plonk-arithmetization.md
- https://nmohnblatt.github.io/zk-jargon-decoder/definitions/arithmetization.html
