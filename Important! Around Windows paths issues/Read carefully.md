For simplicity and time saving, do not use regexes (in production)

- Square brackets and -Path patterns
- Only use LiteralPath
- Long path in Excel and office apps 

Two tier fallback used in pptx conversion:

Also: paths are case insensitive, while JAVA is case sensitive


--------------------------------------------------
If path contains square brackets, the file is not found, but it's there.
```
PDF not found at: D:\files-datastore\SillenoProjectControl-Mirror\04_Periodic Reporting\05_Еженедельная отчетность\Аппаратное совещание\(Файлы для заполнения отделами здесь)\__spo_store\[PROCUREMENT]_Silleno_Weekly_Meeting_Presentation.pptx.rendered\f72c4ec7_v495.0_[PROCUREMENT]_Silleno_Weekly_Meeting_Presentation.pptx.pdf
```
------------------------------


===============================================================
  Conversion failed after 1,3s - PDF not found at: D:\files-datastore\SillenoProjectControl-Mirror\06_Project Control General\03_HR\250730_КПД сотрудников ПК на 2пг2025\Департамент КСПиОК\Отдел контроля изменений\Координатор по КИ - оценка КПД за 2 ПГ 2025\5 NPV procedure 20%\__spo_store\Приложение 1 - Пофакторный анализ 08082025 [Автосохраненный].pptx.rendered\6ab7ef45_v001.0_Приложение 1 - Пофакторный анализ 08082025 [Автосохраненный].pptx.pdf
  Removed subst: X: