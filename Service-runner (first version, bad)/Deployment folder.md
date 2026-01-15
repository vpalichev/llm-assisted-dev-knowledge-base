Yes, there is **one deployment folder** in your development project, but here's the key concept:

## How it works:

### In Development (your machine):

```
d:\projects\service-runner-v01\
├── src/                           ← Your source code
├── management-scripts/            ← Build tools
├── winsw/                         ← WinSW config
├── config.edn                     ← Dev config
└── deployment/                    ← GENERATED package
    ├── service-runner.exe         ← Ready to deploy
    ├── service-runner.jar         ← Your app
    ├── service-runner.xml         ← WinSW config
    ├── config.edn                 ← Copy of config
    ├── install-service.bat        ← Install script
    └── ...                        ← Everything needed
```

### On Production Server (after copying):

```
C:\ServiceRunner\                  ← You copy deployment/ here
├── service-runner.exe             ← Same files
├── service-runner.jar
├── service-runner.xml
├── config.edn
├── install-service.bat
└── ...
```

## So there is:

1. **One source** = `d:\projects\service-runner-v01\deployment\` (on your dev machine)
2. **Many possible destinations** = You can copy this folder to multiple servers

## The workflow:

1. You run `management-scripts\build-and-package.bat` **once** on your dev machine
2. This creates/updates `deployment/` folder with everything needed
3. You **copy** the `deployment/` folder to production server(s)
4. You can copy it to `C:\ServiceRunner\`, `D:\Apps\ServiceRunner\`, or anywhere else
5. You can deploy to multiple servers from the same build

**Important**: The `deployment/` folder is **regenerated** each time you run `build-and-package.bat`. It's not meant to be edited directly - it's the output of the build process.