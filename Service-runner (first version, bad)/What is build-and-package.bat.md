## What is `build-and-package.bat`?

It's a **Windows batch script** that automates the complete build and deployment packaging process for the Service Runner application. Think of it as a "one-click deployment package creator."

## Why do you need it?

Without this script, you would have to manually:

1. Build the Clojure application into a JAR file
2. Copy the JAR to the right location
3. Copy WinSW files
4. Copy configuration files
5. Copy management scripts
6. Create the correct directory structure
7. Ensure everything is named correctly

This script does all of that automatically in **9 automated steps**.

## What does it do?

### Step-by-step breakdown:

1. **[1/9] Checks prerequisites**
    
    - Verifies WinSW.exe exists
    - Verifies Clojure CLI is installed
    - Verifies config.edn exists
2. **[2/9] Cleans old builds**
    
    - Removes previous deployment directory contents
    - Cleans the target directory
3. **[3/9] Builds the uberjar**
    
    - Runs `clj -T:build uber` to compile your Clojure code into a single executable JAR file
    - This JAR contains all dependencies bundled together
4. **[4/9] Verifies the build**
    
    - Checks that `service-runner.jar` was created
    - Shows the JAR file size
5. **[5/9] Creates deployment structure**
    
    - Creates `deployment/` directory
    - Creates subdirectories (`logs/`, `example-scripts/`)
6. **[6/9] Copies the JAR**
    
    - Copies the built JAR to the deployment folder
7. **[7/9] Copies WinSW files**
    
    - Copies `WinSW.exe` → renamed to `service-runner.exe`
    - Copies `service-runner.xml` (WinSW configuration)
8. **[8/9] Copies configs and scripts**
    
    - Copies `config.edn`
    - Copies all management scripts (install-service.bat, etc.)
    - Copies example scripts
9. **[9/9] Creates documentation**
    
    - Generates a `README.txt` in the deployment folder explaining what's inside

## What you get at the end:

A complete, self-contained **deployment** folder that you can:

- Copy to any Windows server
- Run `install-service.bat` as Administrator
- Have a fully working Windows service

## Why is this important?

**Portability**: The deployment folder is a complete package that can be zipped and deployed to production servers without needing the development environment (no Clojure CLI, no source code needed).

**Consistency**: Everyone gets the same deployment structure every time.

**Simplicity**: One command creates everything needed for production deployment.