1. Don't forget to disable drag and drop, but.... 
2. To refresh $PATH variables use refresenvsomething.exe
3. Fix the specification for Converting Office Files to Web: it wrongly takes Basename for output file name, not full name













## DLP Detection Results

**Found: GTB Technologies Endpoint Protector 16.4.0.52061**

Key processes running:

|Process|Count|Purpose|
|---|---|---|
|GTBOCRWorker64|16|OCR content scanning|
|GTBScanner|1|File scanning|
|GTBInjector32/64|2|File access hooks|
|GTBDiscoveryAgent64|1|Discovery agent|

**This is likely causing the slow conversions** - GTB DLP performs OCR content inspection on Office files, which can take 30-60 seconds for complex documents.