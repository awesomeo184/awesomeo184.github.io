---  
emoji: 📝  
title: '애플리케이션 레이어 테스트는 통합테스트를 해야한다고 생각하는 이유'   
date: '2022-05-01 23:00:00'  
author: 어썸오  
tags: test
categories: OOP&TEST
---  

우테코 레벨 2 과정을 진행하면서 테스트에 대한 고민을 많이 했습니다. 레벨 1에서는 주로 콘솔에서 돌아가는 애플리케이션을 작성했기 때문에 거의 도메인에 대한 테스트만 작성했습니다.

레벨 2에서는 웹 애플리케이션을 만들었고, 미션 대부분에서 Layered Architecture를 사용했는데요. presentation  layer와 service(application) layer를 어떻게 테스트할 것인가가 주요 관심사였습니다. 

단위테스트와 통합테스트

