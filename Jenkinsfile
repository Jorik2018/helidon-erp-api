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

stage('Configure Production') {
    steps {
        powershell '''
            $config = "$env:WORKSPACE\\src\\main\\resources\\application.yaml"

            Write-Host "Configurando application.yaml para produccion"
            Write-Host "PORT=$env:PORT"

            $content = Get-Content $config -Raw

            $content = $content -replace `
                '(?m)^\\s*port:\\s*\\d+\\s*$', `
                "  port: $env:PORT"

            Set-Content `
                -Path $config `
                -Value $content `
                -Encoding UTF8

            Write-Host "Configuracion actual:"
            Get-Content $config
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
                clean installDist -x test

            if errorlevel 1 (
                echo ERROR: Gradle build failed
                exit /b 1
            )
        '''
    }
}

      stage('Verify Distribution') {
    steps {
        bat '''
            @echo off

            echo ===============================
            echo ===== Verify Distribution =====
            echo ===============================

            if not exist "build\\install\\helidon-erp-api\\lib" (
                echo ERROR: Distribution lib directory not found
                exit /b 1
            )

            if not exist "build\\install\\helidon-erp-api\\bin" (
                echo ERROR: Distribution bin directory not found
                exit /b 1
            )

            echo Distribution OK

            dir "build\\install\\helidon-erp-api\\lib"
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

stage('Deploy Distribution') {
    steps {
        powershell '''
            Write-Host "=============================="
            Write-Host "===== Deploy Distribution ===="
            Write-Host "=============================="

            $source = "$env:WORKSPACE\\build\\install\\helidon-erp-api"
            $destination = $env:DEPLOY_DIR

            if (-not (Test-Path $source)) {
                throw "Distribution not found: $source"
            }

            if (-not (Test-Path $destination)) {
                New-Item `
                    -ItemType Directory `
                    -Path $destination `
                    -Force | Out-Null
            }

            robocopy `
                $source `
                $destination `
                /MIR

            $code = $LASTEXITCODE

            if ($code -gt 7) {
                throw "Robocopy failed with exit code $code"
            }

            Write-Host "Distribution deployed:"
            Write-Host $destination

            exit 0
        '''
    }
}

stage('Configure Service') {
    steps {
        withCredentials([
            string(
                credentialsId: 'MONGO_URI',
                variable: 'MONGODB_URI'
            )
        ]) {
            powershell '''
                Write-Host "=============================="
                Write-Host "===== Configure Service ====="
                Write-Host "=============================="

                $mongoUri = $env:MONGODB_URI

                # Agregar database "pokemon" a la URI de Atlas
                $mongoUri = $mongoUri.Replace(
                    "mongodb.net/?",
                    "mongodb.net/pokemon?"
                )

                if ($mongoUri -eq $env:MONGODB_URI) {
                    Write-Host "WARNING: No se encontro mongodb.net/? en MONGODB_URI"
                }
                else {
                    Write-Host "MONGODB_URI ajustado para database pokemon"
                }

                $env:MONGODB_URI_DEPLOY = $mongoUri

                & "$env:PYTHON_HOME\\python.exe" `
                    "$env:SERVICE_MANAGER" `
                    install `
                    "$env:SERVICE_ID" `
                    "$env:DEPLOY_DIR" `
                    --name "$env:SERVICE_NAME" `
                    --description "$env:SERVICE_DESCRIPTION" `
                    --type rust `
                    --executable "$env:JAVA_27_HOME\\bin\\java.exe" `
                    --args "-cp lib\\* org.isobit.erp.Main" `
                    --env "MONGODB_URI=$mongoUri"

                if ($LASTEXITCODE -ne 0) {
                    throw "Service configuration failed"
                }
            '''
        }
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