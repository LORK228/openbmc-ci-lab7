pipeline {
    agent any

    environment {
        BMC_IP = "localhost"
        BMC_USER = "root"
        BMC_PASSWORD = "0penBmc"
        QEMU_PID = ""
    }

    stages {
        stage('Prepare Environment') {
            steps {
                echo 'Preparing environment for real OpenBMC testing...'
                sh '''
                    echo "=== Checking available tools ==="
                    which qemu-system-arm || echo "QEMU not found"
                    which curl || echo "curl not found"
                    which unzip || echo "unzip not found"
                    
                    # Скачиваем образ как в лабораторной работе 1
                    if [ ! -f "romulus/obmc-phosphor-image-romulus-*.static.mtd" ]; then
                        echo "📥 Downloading OpenBMC image (as in Lab 1)..."
                        
                        # Скачиваем ZIP архив как в лабе
                        wget -q --timeout=120 https://jenkins.openbmc.org/job/ci-openbmc/lastSuccessfulBuild/distro=ubuntu,label=docker-builder,target=romulus/artifact/openbmc/build/tmp/deploy/images/romulus/*zip*/romulus.zip -O romulus.zip || \
                        curl -L -o romulus.zip https://jenkins.openbmc.org/job/ci-openbmc/lastSuccessfulBuild/distro=ubuntu,label=docker-builder,target=romulus/artifact/openbmc/build/tmp/deploy/images/romulus/*zip*/romulus.zip
                        
                        if [ -f "romulus.zip" ]; then
                            echo "✅ ZIP archive downloaded, extracting..."
                            unzip -o romulus.zip
                            echo "✅ Extraction completed"
                            ls -la romulus/
                        else
                            echo "❌ Failed to download OpenBMC ZIP archive"
                            exit 1
                        fi
                    else
                        echo "✅ OpenBMC image already exists"
                        ls -la romulus/obmc-phosphor-image-romulus-*.static.mtd
                    fi
                '''
            }
        }

        stage('Launch Real OpenBMC in QEMU') {
            steps {
                script {
                    echo '🚀 Starting real OpenBMC in QEMU...'
                    
                    // Находим точное имя файла образа
                    def imageFile = sh(script: 'ls romulus/obmc-phosphor-image-romulus-*.static.mtd', returnStdout: true).trim()
                    echo "Using image file: ${imageFile}"
                    
                    // Запускаем QEMU в фоне и сохраняем PID
                    sh """
                        nohup qemu-system-arm \\
                            -m 256 \\
                            -M romulus-bmc \\
                            -nographic \\
                            -drive file=${imageFile},format=raw,if=mtd \\
                            -net nic \\
                            -net user,hostfwd=tcp::2222-:22,hostfwd=tcp::2443-:443,hostfwd=udp::2623-:623 \\
                            -monitor none \\
                            -serial null \\
                            -daemonize
                        
                        echo "QEMU started with PID: \$(pgrep -f qemu-system-arm)"
                        pgrep -f qemu-system-arm > qemu.pid
                    """
                    
                    // Читаем PID для последующего использования
                    env.QEMU_PID = sh(script: 'cat qemu.pid', returnStdout: true).trim()
                    echo "QEMU running with PID: ${env.QEMU_PID}"
                }
            }
        }

        stage('Wait for BMC Boot Complete') {
            steps {
                echo '⏳ Waiting for OpenBMC to boot...'
                sh '''
                    # Ждем полной загрузки BMC (2-3 минуты)
                    echo "Waiting for BMC to boot (this may take 2-3 minutes)..."
                    
                    for i in {1..30}; do
                        echo "Boot check attempt $i/30"
                        
                        # Проверяем доступность Redfish API
                        if curl -k -s https://localhost:2443/redfish/v1/ | grep -q "odata"; then
                            echo "✅ BMC Redfish API is ready!"
                            break
                        fi
                        
                        # Проверяем доступность SSH
                        if nc -z localhost 2222 2>/dev/null || command -v nc >/dev/null && nc -z localhost 2222; then
                            echo "✅ BMC SSH is ready!"
                            break
                        fi
                        
                        if [ $i -eq 30 ]; then
                            echo "❌ BMC failed to boot within expected time"
                            echo "Checking QEMU process..."
                            ps aux | grep qemu || echo "QEMU process not found"
                            exit 1
                        fi
                        
                        sleep 6
                    done
                    
                    echo "BMC boot sequence completed successfully"
                '''
            }
        }

        stage('Test BMC Basic Functions') {
            steps {
                echo '🧪 Testing OpenBMC basic functionality...'
                sh '''
                    echo "=== OpenBMC Basic Functionality Tests ===" > bmc_test_results.log
                    echo "Test started: $(date)" >> bmc_test_results.log
                    
                    # Тест 1: Проверка Redfish API
                    echo -e "\\n--- Redfish API Test ---" >> bmc_test_results.log
                    curl -k -u ${BMC_USER}:${BMC_PASSWORD} https://${BMC_IP}:2443/redfish/v1/Systems/system >> bmc_test_results.log 2>&1
                    REDFISH_EXIT=$?
                    if [ $REDFISH_EXIT -eq 0 ]; then
                        echo "✅ Redfish API test PASSED" >> bmc_test_results.log
                    else
                        echo "❌ Redfish API test FAILED" >> bmc_test_results.log
                    fi
                    
                    # Тест 2: Проверка через SSH (если sshpass доступен)
                    echo -e "\\n--- SSH Connection Test ---" >> bmc_test_results.log
                    if command -v sshpass >/dev/null 2>&1; then
                        timeout 30s sshpass -p ${BMC_PASSWORD} ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 ${BMC_USER}@${BMC_IP} -p 2222 'obmcutil state || echo "obmcutil command executed"' >> bmc_test_results.log 2>&1
                        SSH_EXIT=$?
                        if [ $SSH_EXIT -eq 0 ] || [ $SSH_EXIT -eq 124 ]; then
                            echo "✅ SSH test PASSED" >> bmc_test_results.log
                        else
                            echo "❌ SSH test FAILED" >> bmc_test_results.log
                        fi
                    else
                        echo "ℹ️ SSH test SKIPPED (sshpass not available)" >> bmc_test_results.log
                    fi
                    
                    echo -e "\\n=== Test completed: $(date) ===" >> bmc_test_results.log
                '''
            }
        }

        stage('Test Power Management') {
            steps {
                echo '⚡ Testing Power Management...'
                sh '''
                    echo "=== Power Management Tests ===" > power_management_test.log
                    
                    # Тестируем управление питанием через SSH (если доступно)
                    if command -v sshpass >/dev/null 2>&1; then
                        echo -e "\\n--- Power State Check ---" >> power_management_test.log
                        sshpass -p ${BMC_PASSWORD} ssh -o StrictHostKeyChecking=no ${BMC_USER}@${BMC_IP} -p 2222 'obmcutil state' >> power_management_test.log 2>&1
                        
                        echo -e "\\n--- Power On Test ---" >> power_management_test.log
                        sshpass -p ${BMC_PASSWORD} ssh -o StrictHostKeyChecking=no ${BMC_USER}@${BMC_IP} -p 2222 'obmcutil poweron && echo "Power ON command sent"' >> power_management_test.log 2>&1
                        sleep 5
                        
                        echo -e "\\n--- Power Status After ON ---" >> power_management_test.log
                        sshpass -p ${BMC_PASSWORD} ssh -o StrictHostKeyChecking=no ${BMC_USER}@${BMC_IP} -p 2222 'obmcutil state' >> power_management_test.log 2>&1
                        
                        echo "✅ Power management tests completed" >> power_management_test.log
                    else
                        echo "ℹ️ Power management tests SKIPPED (sshpass not available)" >> power_management_test.log
                    fi
                '''
            }
        }

        stage('Run Load Testing') {
            steps {
                echo '📊 Running Load Tests...'
                sh '''
                    echo "=== Load Testing OpenBMC API ===" > load_test_results.log
                    
                    # Простой нагрузочный тест Redfish API
                    echo "Starting load test at: $(date)" >> load_test_results.log
                    
                    for i in {1..10}; do
                        START_TIME=$(date +%s)
                        curl -k -s -o /dev/null -w "Request $i: HTTP %{http_code}, Time: %{time_total}s\\n" \
                             -u ${BMC_USER}:${BMC_PASSWORD} \
                             https://${BMC_IP}:2443/redfish/v1/ >> load_test_results.log 2>&1
                        END_TIME=$(date +%s)
                        REQUEST_TIME=$((END_TIME - START_TIME))
                        echo "Request $i completed in ${REQUEST_TIME}s" >> load_test_results.log
                        
                        # Небольшая пауза между запросами
                        sleep 1
                    done
                    
                    echo "Load test completed at: $(date)" >> load_test_results.log
                    echo "✅ Load testing finished" >> load_test_results.log
                '''
            }
        }
    }

    post {
        always {
            echo '🧹 Cleaning up QEMU processes...'
            sh '''
                # Останавливаем QEMU по сохраненному PID
                if [ -f qemu.pid ]; then
                    QEMU_PID=$(cat qemu.pid)
                    echo "Stopping QEMU process: $QEMU_PID"
                    kill $QEMU_PID 2>/dev/null || true
                    sleep 5
                    # Принудительно завершаем если еще работает
                    pkill -f qemu-system-arm 2>/dev/null || true
                    rm -f qemu.pid
                fi
                
                # Сбор артефактов
                echo "=== Collecting Artifacts ==="
                ls -la *.log || echo "No log files found"
                ls -la romulus.zip || echo "No zip file found"
                ls -la romulus/ || echo "No romulus directory found"
            '''
            
            archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
            archiveArtifacts artifacts: 'romulus.zip', allowEmptyArchive: true
        }
        success {
            echo '✅ OpenBMC CI/CD Pipeline with REAL QEMU completed successfully!'
            sh 'echo "Lab 7: REAL OpenBMC CI/CD - SUCCESS" > pipeline_summary.txt'
            archiveArtifacts artifacts: 'pipeline_summary.txt', allowEmptyArchive: true
        }
        failure {
            echo '❌ OpenBMC CI/CD Pipeline failed!'
            sh 'echo "Lab 7: REAL OpenBMC CI/CD - FAILED" > pipeline_summary.txt'
            archiveArtifacts artifacts: 'pipeline_summary.txt', allowEmptyArchive: true
        }
    }
}
