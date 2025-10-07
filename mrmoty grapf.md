

![[Pasted image 20251003134631.png]]


 React Native “Performance Monitor”

- **RAM 397.54 MB** – približna ukupna memorija procesa app-a (na simulatoru ume da bude naduvano).
    
- **JSC 0.00 MB** – veličina heap-a JavaScriptCore-a. Pošto koristiš **Hermes**, ovde stoji 0 (taj broj je relevantan samo kad je engine JSC).
    
- **Views 182 / 232** – broj čvorova u hijerarhiji:
    
    - **gornji**: približno broj **montiranih native UIViews**,
        
    - **donji**: broj **Shadow/ Yoga čvorova** (layout n) – obično veći.
        
- **UI 58** i **JS 60** (sive kutije) – **FPS** niti: UI nit i JS nit. Više je bolje, target je ~60fps.