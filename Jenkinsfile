pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

environment {
    SERVICE_ID = 'helidon-erp-api'
    SERVICE_NAME = 'Helidon ERP API'
    SERVICE_DESCRIPTION = 'Helidon ERP API'

    DEPLOY_DIR = 'D:\\microservices\\helidon-erp-api'
    PORT = '2267'

    JAVA_HOME = 'D:\\java\\jdk-17.0.5+8'
    JAVA_27_HOME = 'D:\\java\\jdk-27'
    

    PATH = "${JAVA_HOME}\\bin;${env.PATH}"
}

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }
stage('Install Java 27') {
    steps {
        powershell '''
            $javaHome = "D:\\java\\jdk-27"
            $javaExe = "$javaHome\\bin\\java.exe"

            if (Test-Path $javaExe) {
                Write-Host "Java 27 ya esta instalado:"
                & $javaExe -version
                exit 0
            }

            Write-Host "Java 27 no encontrado. Instalando..."

            [Net.ServicePointManager]::SecurityProtocol = `
                [Net.SecurityProtocolType]::Tls12

            $javaRoot = "D:\\java"
            $zip = "$env:TEMP\\jdk27.zip"
            $extractDir = "$env:TEMP\\jdk27-extract"

            if (-not (Test-Path $javaRoot)) {
                New-Item `
                    -ItemType Directory `
                    -Path $javaRoot `
                    -Force | Out-Null
            }

            if (Test-Path $extractDir) {
                Remove-Item `
                    $extractDir `
                    -Recurse `
                    -Force
            }

            New-Item `
                -ItemType Directory `
                -Path $extractDir `
                -Force | Out-Null

            if (Test-Path $zip) {
                Remove-Item $zip -Force
            }

            Write-Host "Descargando Temurin JDK 27..."

            $url = "https://api.adoptium.net/v3/binary/latest/27/ga/windows/x64/jdk/hotspot/normal/eclipse"

            Invoke-WebRequest `
                -UseBasicParsing `
                -Uri $url `
                -OutFile $zip

            if (-not (Test-Path $zip)) {
                throw "No se pudo descargar JDK 27"
            }

            Write-Host "ZIP descargado:"
            Write-Host $zip

            Write-Host "Descomprimiendo..."

            Expand-Archive `
                -Path $zip `
                -DestinationPath $extractDir `
                -Force

            $jdkDir = Get-ChildItem `
                -Path $extractDir `
                -Directory |
                Select-Object -First 1

            if (-not $jdkDir) {
                throw "No se encontro el directorio del JDK dentro del ZIP"
            }

            Write-Host "Directorio encontrado:"
            Write-Host $jdkDir.FullName

            if (Test-Path $javaHome) {
                Remove-Item `
                    $javaHome `
                    -Recurse `
                    -Force
            }

            Move-Item `
                -Path $jdkDir.FullName `
                -Destination $javaHome

            Remove-Item `
                $zip `
                -Force `
                -ErrorAction SilentlyContinue

            Remove-Item `
                $extractDir `
                -Recurse `
                -Force `
                -ErrorAction SilentlyContinue

            if (-not (Test-Path $javaExe)) {
                throw "Java 27 no se instalo correctamente"
            }

            Write-Host "================================="
            Write-Host "Java 27 instalado correctamente"
            Write-Host "================================="

            & $javaExe -version
        '''
    }
}
        stage('Environment') {
            steps {
                bat '''
                    @echo off

                    echo ============================
                    echo ===== Environment =========
                    echo ============================

                    echo.
                    echo ===== JAVA =====
                    where java
                    java -version

                    if errorlevel 1 (
                        echo ERROR: Java not found
                        exit /b 1
                    )

                    echo.
                    echo ===== GIT =====
                    where git
                    git --version

                    if errorlevel 1 (
                        echo ERROR: Git not found
                        exit /b 1
                    )

                    echo.
                    echo ===== PROJECT =====

                    if not exist build.gradle (
                        if not exist build.gradle.kts (
                            echo ERROR: build.gradle or build.gradle.kts not found
                            exit /b 1
                        )
                    )

                    if not exist gradlew.bat (
                        echo ERROR: gradlew.bat not found
                        exit /b 1
                    )
                '''
            }
        }

stage('Gradle Version') {
    steps {
        bat '''
            @echo off

            echo ============================
            echo ===== Gradle ===============
            echo ============================

            echo Gradle JVM:
            "%JAVA_HOME%\\bin\\java.exe" -version

            echo.
            echo Java 27 toolchain:
            "%JAVA_27_HOME%\\bin\\java.exe" -version

            echo.
            call gradlew.bat --version

            if errorlevel 1 (
                echo ERROR: Gradle wrapper failed
                exit /b 1
            )
        '''
    }
}

stage('Build') {
    steps {
        bat '''
            @echo off

            echo ============================
            echo ===== Build ================
            echo ============================

            call gradlew.bat ^
                -Porg.gradle.java.installations.paths="%JAVA_27_HOME%" ^
                clean build -x test

            if errorlevel 1 (
                echo ERROR: Gradle build failed
                exit /b 1
            )
        '''
    }
}

        stage('Verify Artifact') {
            steps {
                powershell '''
                    Write-Host "============================"
                    Write-Host "===== Verify Artifact ======="
                    Write-Host "============================"

                    $jars = Get-ChildItem `
                        -Path "$env:WORKSPACE\\build\\libs" `
                        -Filter "*.jar" `
                        -File |
                        Where-Object {
                            $_.Name -notmatch "-plain\\.jar$"
                        }

                    if (-not $jars) {
                        throw "No executable JAR found in build\\libs"
                    }

                    Write-Host "Artifacts encontrados:"

                    foreach ($jar in $jars) {
                        Write-Host " - $($jar.FullName)"
                    }
                '''
            }
        }

        stage('Stop Service') {
            steps {
                bat '''
                    @echo off

                    echo ============================
                    echo ===== Stop Service =========
                    echo ============================

                    "%PYTHON_HOME%\\python.exe" ^
                        "%SERVICE_MANAGER%" ^
                        stop ^
                        "%SERVICE_ID%"

                    exit /b 0
                '''
            }
        }

        stage('Deploy Files') {
            steps {
                powershell '''
                    $source = $env:WORKSPACE
                    $destination = $env:DEPLOY_DIR

                    Write-Host "============================"
                    Write-Host "===== Deploy Files =========="
                    Write-Host "============================"

                    Write-Host "SOURCE: $source"
                    Write-Host "DEST:   $destination"

                    if (-not (Test-Path $destination)) {
                        New-Item `
                            -ItemType Directory `
                            -Path $destination `
                            -Force | Out-Null
                    }

                    robocopy `
                        $source `
                        $destination `
                        /MIR `
                        /XD ".git" ".gradle" "build" `
                        /XF "*.log"

                    $code = $LASTEXITCODE

                    if ($code -gt 7) {
                        throw "Robocopy failed with exit code $code"
                    }

                    exit 0
                '''
            }
        }

        stage('Deploy Artifact') {
            steps {
                powershell '''
                    Write-Host "============================"
                    Write-Host "===== Deploy Artifact ======="
                    Write-Host "============================"

                    $jar = Get-ChildItem `
                        -Path "$env:WORKSPACE\\build\\libs" `
                        -Filter "*.jar" `
                        -File |
                        Where-Object {
                            $_.Name -notmatch "-plain\\.jar$"
                        } |
                        Select-Object -First 1

                    if (-not $jar) {
                        throw "Executable JAR not found"
                    }

                    $target = "$env:DEPLOY_DIR\\helidon-erp-api.jar"

                    Copy-Item `
                        $jar.FullName `
                        $target `
                        -Force

                    Write-Host "Artifact deployed:"
                    Write-Host $target
                '''
            }
        }

  stage('Configure Service') {
    steps {
        bat '''
            @echo off

            echo ==============================
            echo ===== Configure Service =====
            echo ==============================

            set "JAVA_EXE=%JAVA_27_HOME%\\bin\\java.exe"

            if not exist "%JAVA_EXE%" (
                echo ERROR: Java executable not found:
                echo %JAVA_EXE%
                exit /b 1
            )

            "%PYTHON_HOME%\\python.exe" ^
                "%SERVICE_MANAGER%" ^
                install ^
                "%SERVICE_ID%" ^
                "%DEPLOY_DIR%" ^
                --name "%SERVICE_NAME%" ^
                --description "%SERVICE_DESCRIPTION%" ^
                --type rust ^
                --executable "%JAVA_EXE%" ^
                --args "-jar helidon-erp-api.jar" ^
                --env "SERVER_PORT=%PORT%"

            if errorlevel 1 (
                echo ERROR: Service configuration failed
                exit /b 1
            )
        '''
    }
}

        stage('Start Service') {
            steps {
                bat '''
                    @echo off

                    echo ============================
                    echo ===== Start Service ========
                    echo ============================

                    "%PYTHON_HOME%\\python.exe" ^
                        "%SERVICE_MANAGER%" ^
                        start ^
                        "%SERVICE_ID%"

                    if errorlevel 1 (
                        echo ERROR: Could not start service
                        exit /b 1
                    )
                '''
            }
        }

        stage('Verify Service') {
            steps {
                bat '''
                    @echo off

                    echo ============================
                    echo ===== Verify Service ======
                    echo ============================

                    sc query "%SERVICE_ID%"

                    sc query "%SERVICE_ID%" | findstr /I "RUNNING"

                    if errorlevel 1 (
                        echo ERROR: Helidon service is not running
                        exit /b 1
                    )

                    echo.
                    echo ===== Port %PORT% =====

                    netstat -ano | findstr ":%PORT%"

                    if errorlevel 1 (
                        echo ERROR: Helidon API is not listening on port %PORT%
                        exit /b 1
                    )
                '''
            }
        }

        stage('Health Check') {
            steps {
                powershell '''
                    $url = "http://127.0.0.1:$env:PORT/health"
                    $maxAttempts = 10

                    for (
                        $attempt = 1;
                        $attempt -le $maxAttempts;
                        $attempt++
                    ) {

                        Write-Host "===================================="
                        Write-Host "Health check $attempt/$maxAttempts"
                        Write-Host "GET $url"

                        try {

                            $response = Invoke-WebRequest `
                                -UseBasicParsing `
                                -Uri $url `
                                -TimeoutSec 5

                            Write-Host "HTTP: $($response.StatusCode)"

                            if ($response.StatusCode -eq 200) {
                                Write-Host "Helidon ERP API OK"
                                exit 0
                            }

                        }
                        catch {
                            Write-Host $_.Exception.Message
                        }

                        if ($attempt -lt $maxAttempts) {
                            Start-Sleep -Seconds 3
                        }
                    }

                    throw "Helidon ERP API failed health check."
                '''
            }
        }

    }

    post {

        success {
            echo 'Helidon ERP API deployed successfully.'
            echo 'Internal URL: http://127.0.0.1:2267/'
            echo 'Health URL: http://127.0.0.1:2267/health'
        }

        failure {
            echo 'Helidon ERP API deployment FAILED.'
        }

    }
}