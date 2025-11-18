pipeline {
    agent any

    stages {
        stage('Launch QEMU with OpenBMC') {
            steps {
                script {
                    // Запускаем QEMU в фоне. Это сложный шаг, который может потребовать особой настройки Jenkins-агента.
                    sh 'nohup qemu-system-arm -m 256 -M romulus-bmc -nographic -drive file=./obmc-image.mtd,format=raw,if=mtd -net nic -net user,hostfwd=:0.0.0.0:2222-:22,hostfwd=:0.0.0.0:2443-:443,hostfwd=udp:0.0.0.0:2623-:623,hostname=qemu > qemu.log 2>&1 &'
                    // Ждем, пока BMC загрузится
                    sh 'sleep 60'
                }
            }
        }

        stage('Run OpenBMC Autotests') {
            steps {
                sh '''
                    echo "=== Running Real BMC Tests ==="
                    # Предполагается, что obmcutil и ipmitool доступны, а QEMU запущен
                    sshpass -p '0penBmc' ssh -o StrictHostKeyChecking=no root@localhost -p 2222 'obmcutil state' > bmc_state.log
                    sshpass -p '0penBmc' ssh -o StrictHostKeyChecking=no root@localhost -p 2222 'obmcutil poweron' > power_control.log
                    ipmitool -I lanplus -H localhost -p 2623 -U root -P 0penBmc chassis status > ipmi_status.log
                    curl -k -u root:0penBmc https://localhost:2443/redfish/v1/Systems/system > redfish_system.log
                '''
            }
            post {
                always {
                    // Сохраняем логи тестов как артефакты
                    archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
                }
            }
        }

        stage('Run WebUI Tests') {
            steps {
                // Предполагается, что в репозитории есть скрипт для Selenium/Playwright
                sh 'python3 run_webui_tests.py'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'webui_test_report.html, selenium_screenshots/*', allowEmptyArchive: true
                }
            }
        }

        stage('Run Load Testing') {
            steps {
                // Запускаем Locust в режиме одного запуска (без Web UI)
                sh 'locust -f locustfile.py --headless -u 10 -r 2 --run-time 1m --html=locust_report.html'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'locust_report.html', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        always {
            // Останавливаем QEMU в конце пайплайна
            sh 'pkill -f qemu-system-arm || true'
            echo 'Pipeline execution finished.'
        }
    }
}
