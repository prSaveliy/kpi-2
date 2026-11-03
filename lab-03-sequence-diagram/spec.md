# spec.md - Sequence Diagrams

## Намір
Створити sequence-діаграму на кожен сценарій з обраного use case у синтаксисі Mermaid sequenceDiagram.

## Обраний Use Case:
- `UC-04` (Зареєструватися в індивідуальному турнірі) із включенням `UC-06` (Оплатити турнірний внесок)

## Актори
- Player
- Payment Gateway

## Компоненти/Об'єкти
- Frontend (WebUI)
- RegistrationController
- RegistrationService
- PaymentService
- Database


# Критерії прийняття
- [ ] Sequence-діаграма для кожного сценарію у синтаксисі Mermaid sequenceDiagram, детально; у кожній — щонайменше один фрагмент alt/opt/loop - вивід в model/
- [ ] Рендер кожної діаграми у .png - вивід в render/
- [ ] Activation bar учасника, що працює поза фрагментом або в кількох гілках, починається до і закінчується після умовного фрагмента — не обривається в одній гілці
- [ ] Узгодженість з ER-діграмою з lab 1 [model.mmd](../lab-01-er/model.mmd)
- [ ] Узгодженість кроків з сценаріями Gherkin [requirements.md](../lab-02-requirements-usecases/requirements.md)
- [ ] Читабельність діаграм в render/ - немає зловживання стрілок поверх стрілок, незаплутані шляхи, намеє накладання інформації з неможливістю прочитати.