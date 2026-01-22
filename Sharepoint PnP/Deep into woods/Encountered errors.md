# SharePoint Download Errors - Categorized

## Large File Timeout Issues

### Response Ended Prematurely (50+ MB)
```
Failed to download: /sites/SillenoProjectControl/Shared Documents/01_Cost Estimation and Control/02. Ценообразование/01. Пиролиз/10. Acceptance & Invoicing/2025/11. Nov'25 РРС 08,09,10/PPC09_28.11/Final_19.12 ПДФ РРС09/Consolidated/PPC09 for November'25.pdf - The response ended prematurely, with at least 18556551 additional bytes expected. (ResponseEnded)
```

### HttpClient Timeout (100 seconds)
```
Downloading: /sites/SillenoProjectControl/Shared Documents/02_Contracts/02_EPC/02_PEU, UI&O/06_Reporting/Monthly/2025 12/Attachments/SLN-2725-0000-04-R-0010 _A_Att.10.05. LEVEL III SCHEDULE – CONTROL LEVEL SCHEDULE.pdf (v1.0, 112 MB)
Failed to download: The request was canceled due to the configured HttpClient.Timeout of 100 seconds elapsing.
```

## File Not Found Errors

### Russian Filename Files
```
Download current version to d:\files-datastore\SillenoProjectControl-Mirror\01_Cost Estimation and Control\20. Contract Cost Control\3. Others\3. Contracts\10. Author Superv. (ECU, PE, UI&O) - АО КИНГ\Milestones for KING (Last Version) Rev4 к ДС#1 от 201225.xlsx failed (attempt 3/3): Exception calling "ExecuteQuery" with "0" argument(s): "Файл не найден."
```
```
Download current version to d:\files-datastore\SillenoProjectControl-Mirror\01_Cost Estimation and Control\22. Материалы к КПД 2 го полугодия 2025г\Согласован состав работ по 30% 3DModel BOQ_PE UIO\PEU-JVTS-SLN-MOM-00206-PCO.xlsx failed (attempt 3/3): Exception calling "ExecuteQuery" with "0" argument(s): "Файл не найден."
```

### BBB Source Files (30% Documentation)
```
Download current version to d:\files-datastore\SillenoProjectControl-Mirror\02_Contracts\02_EPC\02_PEU, UI&O\07_Important documents\BBB\30%\SLN-2725-0000-70-B-0002_B_2_source.xlsx failed (attempt 3/3): Exception calling "ExecuteQuery" with "0" argument(s): "Файл не найден."
```

### Weekly Report Files - Temporary Building
```
Download current version to d:\files-datastore\SillenoProjectControl-Mirror\02_Contracts\03_Other Contracts\02_Temporary building\06_Reporting\02_Weekly report\2025\50_Weekly report #50\Temporary building - WR 12.12.2025 (Analys).xlsx failed (attempt 3/3): Exception calling "ExecuteQuery" with "0" argument(s): "Файл не найден."
```
```
Download current version to d:\files-datastore\SillenoProjectControl-Mirror\02_Contracts\03_Other Contracts\02_Temporary building\06_Reporting\02_Weekly report\2025\52_Weekly report #52\МТО W52.xlsx failed (attempt 3/3): Exception calling "ExecuteQuery" with "0" argument(s): "Файл не найден."
```

### Weekly Report Files - Site Preparation
```
Download current version to d:\files-datastore\SillenoProjectControl-Mirror\02_Contracts\03_Other Contracts\01_Site Preparation\06_Reporting\02_Weekly report\02_2025\48_Weekly report #48\Site Preparation - Weekly report 28.11.25.xlsx failed (attempt 3/3): Exception calling "ExecuteQuery" with "0" argument(s): "Файл не найден."
```

## URL Encoding Issues

### Fullwidth Parentheses (U+FF08, U+FF09)
```
Download current version to d:\files-datastore\SillenoProjectControl-Mirror\06_Project Control General\02_General Meetings\Supervisory Board (НС)\Вехи и КПД\2025 2п-е Исполнение\КПД\4. Финансовый директор\5. Получение предложений (RFP) по долгосрочному финансированию\Project Karabatan - Bank Responses to Financing Structure_2025 September%09 vF.xlsx failed (attempt 1/3): Файл /sites/SillenoProjectControl/Shared Documents/06_Project Control General/02_General Meetings/Supervisory Board (НС)/Вехи и КПД/2025 2п-е Исполнение/КПД/4. Финансовый директор/5. Получение предложений (RFP) по долгосрочному финансированию/Project Karabatan - Bank Responses to Financing Structure_2025 September  vF.xlsx не существует.
```

### Percent-Encoded Spaces in Filename
```
Download current version to d:\files-datastore\SillenoProjectControl-Mirror\04_Periodic Reporting\06_Рабочие файлы\Жанузак\COMMON%20PERSONNEL%20HISTORGAMM%20new (version 1).xlsb.xlsx failed (attempt 3/3): Файл /sites/SillenoProjectControl/Shared Documents/04_Periodic Reporting/06_Рабочие файлы/Жанузак/COMMON PERSONNEL HISTORGAMM new (version 1).xlsb.xlsx не существует.
```

## JSON Parsing Errors

### Malformed JSON Stream
```
Get-SharePointAllListItems: D:\projects\Silleno-SharePoint-PnP-Tools-v02\src\Sync-SPOVersions-v02.ps1:72:27
Line |
  72 |  … ointItems = Get-SharePointAllListItems -ListName $sharePointLibraryNa …
     |                ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     | Failed to retrieve list items after retries: Not well formatted JSON stream.
Get-PnPListItem: C:\Users\v.palichev\Documents\PowerShell\Modules\PnP.CustomUtilities\Public\Get-SharePointAllListItems.ps1:95:13
Line |
  95 |              Get-PnPListItem @params
     |              ~~~~~~~~~~~~~~~~~~~~~~~
     | Not well formatted JSON stream.
```

## DNS/Network Errors

### Host Resolution Failures
```
"Timestamp": "2026-01-21T18:18:32.5301091+05:00",
"Type": "VersionFetch",
"FileRef": "/sites/SillenoProjectControl/Shared Documents/04_Periodic Reporting/05_Еженедельная отчетность/Еженедельные отчеты по блокам/2026/02 неделя 2026/UI&O Procurement.xlsx",
"Version": "",
"Message": "No such host is known. (sillenokz.sharepoint.com:443)"
```
```
"Timestamp": "2026-01-21T18:18:54.8040534+05:00",
"Type": "VersionFetch",
"FileRef": "/sites/SillenoProjectControl/Shared Documents/04_Periodic Reporting/05_Еженедельная отчетность/Еженедельные отчеты по блокам/2026/02 неделя 2026/ECU Construction (Status of ECU piling and foundations works).xlsx",
"Version": "",
"Message": "No such host is known. (sillenokz.sharepoint.com:443)"
```