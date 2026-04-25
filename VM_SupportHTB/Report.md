# 📑 Отчет по результатам анализа защищенности: Support
**Статус:** 🔴 КРИТИЧЕСКИЙ УРОВЕНЬ РИСКА

---

## 1. Резюме для руководства
Целью аудита был узел под управлением Windows Server в доменной среде `support.htb`. В ходе тестирования была выявлена цепочка уязвимостей, связанная с избыточными правами владения объектами в Active Directory. Злоумышленник, обладая учетными данными обычного пользователя, может манипулировать атрибутами контроллера домена (DC) для захвата полных прав администратора домена.

**Основные находки:**
*   **Information Disclosure:** Наличие чувствительных данных в общедоступных сетевых ресурсах (SMB).
*   **Insecure AD Control Path:** Наличие права GenericAll у пользователя над объектом контроллера домена.
*   **Resource-Based Constrained Delegation (RBCD):** Возможность компрометации домена через злоупотребление делегированием Kerberos.

---

## 2. Сводка уязвимостей

| ID | Уязвимость |	Вектор | Критичность
| :--- | :--- | :--- | :--- |
| **VULN-01** | Раскрытие учетных данных в SMB-шарах | Удаленный | **HIGH** |
| **VULN-02** | Избыточные ACL права (GenericAll на DC) | Внутренний | **CRITICAL** |
| **VULN-03** | Злоупотребление Kerberos Delegation (RBCD) | Внутренний | **CRITICAL** |

---

## 3. Технический анализ и ход атаки

### 3.1. Внешняя разведка

Первичный анализ портов выявил стандартный набор сервисов Windows/AD (RPC, SMB, LDAP, Kerberos). При перечисления сетевых шар был обнаружен доступ к ресурсу, содержащему информацию о пользователе.

![smbclient_support](screenshots/smdclient_get_support.png)

### 3.2. Первоначальный доступ

С помощью `ldapsearch` были обнаружены учетные данные пользователя `support`. С помощью утилиты `evil-winrm` была установлена удаленная сессия.

**Вектор атаки:**
1. Поиск в шарах: `smbclient //10.129.230.181/support-tools -N`
2. Удаленное подключение: `evil-winrm -i support.htb -u support -p 'Ironside47pleasure40Watchful'`

![evil-winrm_support](screenshots/evilWinRM_support.png)

**Результат: Получен доступ к системе с правами пользователя.*

### 3.3. Анализ векторов управления AD

Анализ графа в BloodHound выявил, что текущий пользователь обладает правом GenericAll над объектом контроллера домена DC.SUPPORT.HTB.

![chain_bloodhound_support](screenshots/chain_bloodhound_support.png)

### 3.4. Повышение привилегий

Используя `impacket-rbcd`, в атрибут контроллера домена `msDS-AllowedToActOnBehalfOfOtherIdentity` был записан подконтрольный компьютер. Это позволило запросить сервисный билет (TGS) от имени Администратора.

![impacket-getST_support](screenshots/getST_support.png)
![impacket-wmiexec_support](screenshots/impacket-wmiexec_support.png)

*Результат: Полная компрометация инфраструктуры.*

### 3.5. Сопоставление с матрицей MITRE ATT&CK

Для классификации действий, предпринятых в ходе аудита, использовалась модель знаний о тактиках и техниках атакующих.

| Тактика | ID | Название техники | Описание действия |
| :--- | :--- | :--- | :--- |
| Discovery | T1046 | Network Service Scanning | Сканирование портов и сервисов (Nmap) |
| Discovery | T1087.002 | Account Discovery: Domain Account | Сбор данных о пользователях через LDAP и SMB |
| Discovery | T1135 | Network Share Discovery | Поиск конфиденциальных данных в сетевых папках |
| Credential Access | T1558 | Steal or Forge Kerberos Tickets | Эксплуатация Kerberos Delegation (RBCD) |
| Privilege Escalation | T1098 | Account Manipulation | Манипуляция атрибутами объекта DC (msDS-AllowedToActOnBehalfOfOtherIdentity) |

---

## 🛡️ 4. План мероприятий по устранению

### 4.1. Первоочередные меры

1. **Ограничение прав в AD:** Удалите избыточные права GenericAll у непривилегированных групп над объектами компьютеров и контроллеров домена.
2. **Защита сетевых ресурсов:** Ограничьте доступ к SMB-шарам и удалите файлы, содержащие учетные данные.

### 4.2. Системная защита

1. **Защищенные пользователи:** Поместите учетные записи администраторов в группу "Protected Users" для блокировки возможности делегирования.
    *   `Add-ADGroupMember -Identity "Protected Users" -Members "Administrator"`
2. **MachineAccountQuota:** Установите значение **ms-DS-MachineAccountQuota** в **0**, чтобы запретить пользователям создавать новые учетные записи компьютеров.
    *   `Set-ADDomain -Identity "support.htb" -Replace @{"ms-DS-MachineAccountQuota"=0}`

### 4.3. Рекомендации по мониторингу

1. Настройте аудит события **ID 4742** для отслеживания изменений в критических атрибутах компьютеров домена.
    *   `auditpol /set /subcategory:"Directory Service Changes" /success:enable /failure:enable`
2. Настройте триггер в SIEM на изменение атрибута **msDS-AllowedToActOnBehalfOfOtherIdentity**.

## 🎯 [Пошаговый пентест](./writeup.md)