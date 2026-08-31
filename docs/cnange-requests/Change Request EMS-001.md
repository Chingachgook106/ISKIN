==========================================================
CR-001 Standard SMB File Exchange — ЗАКРЫТ (CLOSED)
==========================================================
Server (EMS-001 / Beelink5800H, 192.168.0.102):
  Share:      iskin-exchange -> C:\iskin-exchange\{incoming,outgoing,work}
  Access:     only indeez (Full), no Everyone
  Firewall:   File and Printer Sharing (SMB-In) enabled
  Profile:    Private; LanmanServer Running/Automatic
  Reboot:     share, data and permissions persistent

Client (Основной ПК):
  Credential Manager: 192.168.0.102 -> Beelink5800H\indeez (saved)
  Cold reconnect: OK without manual net use
  Drive mapping: persistent UNC / mapped drive
  Known behavior: без saved credentials клиент шлёт локальные (Сергей) 
                  и получает ошибку 1326/86.