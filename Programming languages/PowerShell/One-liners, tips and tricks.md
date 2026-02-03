

#### Enter into PS 7
pwsh.exe




# This info not acquired yet


#### Select all fields: 


#### Convert date to ISO 8601 UTC:

How to filter, normal and through indexer:



Turn indexed object into PSCustomObject


Select field or indexer values


Wrap in count()


list of files and folders:
Get-ChildItem  | Select-Object -ExpandProperty FullName

#### From practice: 
```powershell
Get-ChildItem -File -Recurse | Sort-Object Length -Descending | Select-Object FullName, @{N='Size(MB)';E={[math]::Round($_.Length/1MB,2)}} | Select-Object -First 20
```


#### Use another working folder
Push-Location D:\bin\datomic\datomic-pro-1.0.7469; .\bin\transactor.cmd "d:\TEMP\Testing_new_projects\datomic-connectivity-testing\dev-transactor.properties"; Pop-Location


