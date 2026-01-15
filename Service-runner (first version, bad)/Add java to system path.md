
Add to system path

PS C:\Windows\system32> $javaPath = "C:\Program Files\Microsoft\jdk-21.0.8.9-hotspot\bin"
PS C:\Windows\system32> [Environment]::SetEnvironmentVariable("PATH", [Environment]::GetEnvironmentVariable("PATH", "Machine") + ";$javaPath", "Machine")
PS C:\Windows\system32>





