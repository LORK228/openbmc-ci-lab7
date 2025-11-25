pipeline {
    agent any

    stages {
        stage('Скачиваем образ OpenBMC') {
            steps {
                sh '''
                    if [ ! -f "romulus/obmc-phosphor-image-romulus-"*.static.mtd ]; then
                        wget -q "https://jenkins.openbmc.org/job/ci-openbmc/lastSuccessfulBuild/distro=ubuntu,label=docker-builder,target=romulus/artifact/openbmc/build/tmp/deploy/images/romulus/*zip*/romulus.zip" -O romulus.zip
                        unzip -q romulus.zip
                        rm romulus.zip
                    else
                        echo "Образ уже существует"
                    fi
                '''
            }
        }

        stage('Старт QEMU') {
            steps {
                script {
                    sh '''
                        pkill -f qemu-system-arm || true
                        sleep 1
                        
                        ssh-keygen -f "/var/jenkins_home/.ssh/known_hosts" -R "[localhost]:2222" 2>/dev/null || true
                        IMAGE_FILE=$(ls romulus/obmc-phosphor-image-romulus-*.static.mtd)
                        qemu-system-arm -m 256 -M romulus-bmc -nographic -drive file="$IMAGE_FILE",format=raw,if=mtd -net nic -net user,hostfwd=:0.0.0.0:2222-:22,hostfwd=:0.0.0.0:2443-:443,hostfwd=udp:0.0.0.0:2623-:623,hostname=qemu > qemu.log &
                    '''
                }
                sleep time: 120, unit: 'SECONDS'
                
                sh '''
                    echo "Проверка SSH"
                    if nc -z localhost 2222; then
                        echo "SSH порт доступен"
                        break
                    fi
                    echo "Проверка HTTPS"
                    if nc -z localhost 2443; then
                        echo "HTTPS порт доступен"
                        break
                    fi
                '''
            }
        }

        stage('Запуск Автотеста') {
            steps {
                script {
                    sh '''
                        for i in 1 2 3; do
                            if sshpass -p "0penBmc" ssh -o StrictHostKeyChecking=no -p 2222 root@localhost; then
                                echo "SSH подключение установлено" >> autotests.log
                                sshpass -p "0penBmc" ssh -o StrictHostKeyChecking=no -p 2222 root@localhost '
                                    echo "Память"  
                                    free -h 
                                    echo
                                    echo "Диск"
                                    df -h 
                                    echo
                                    echo "Процессы"
                                    ps aux 
                                    echo
                                    echo "Сетевые интерфейсы"
                                    ip addr
                                    echo
                                    echo "Автотесты завершены"
                                ' >> autotests.log 2>&1
                                break
                            else
                                echo "Провал автотеста" >> autotests.log
                                sleep 3
                            fi
                        done
                        
                        echo "Результат автотестов"
                        cat autotests.log
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'autotests.log', fingerprint: true
                }
            }
        }

        stage('Test BMC WebUI') {
            steps {
                sh '''
                echo "Проверяем доступность WebUI" > webui_test.log
                
                    for i in 1 2 3; do
                        if curl -k -s https://localhost:2443/redfish/v1/; then
                            echo "WebUI Redfish доступен" >> webui_test.log
                            break
                        fi
                        sleep 3
                    done
                    
                    cat > simple_webui_test.sh << 'EOF'
#!/bin/bash

echo "Тест 1 проверка доступности Redfish API"
response=$(curl -k -s -w "%{http_code}" -o /dev/null https://localhost:2443/redfish/v1/)
if [ "$response" -eq 200 ]; then
    echo "Redfish API доступен"
else
    echo "Redfish API недоступен"
    exit 1
fi

echo "Тест 2 получение информации о системе"
curl -k -s https://localhost:2443/redfish/v1/Systems/system

echo "Тест 3 проверка SessionService"
curl -k -s https://localhost:2443/redfish/v1/SessionService

EOF

                    chmod +x simple_webui_test.sh
                    ./simple_webui_test.sh >> webui_test.log
                    echo "WebUI тесты завершены"
                    cat webui_test.log
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'webui_test.log', fingerprint: true
                }
            }

        }
        stage('Run Load Testing') {
            steps {
                script {
                    sh '''
                        echo "Нагрузочное тестирование" > loadtest.log
                        
                        echo "Запускаем простой нагрузочный тест Redfish API" >> loadtest.log
                        
                        for i in 1 2 3; do
                            echo "Запрос $i:" >> loadtest.log
                            start_time=$(date +%s%3N)
                            http_code=$(curl -k -s -w "%{http_code}" -o /dev/null https://localhost:2443/redfish/v1/)
                            end_time=$(date +%s%3N)
                            duration=$((end_time - start_time))
                            echo "  Время: ${duration}ms, Код: $http_code" >> loadtest.log
                            sleep 1
                        done
                        
                        echo "Нагрузочное тестирование завершено" >> loadtest.log
                        cat loadtest.log
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'loadtest.log', fingerprint: true
                }
            }
        }
    }
    post {
        always {
            script {
                sh '''
                    echo "Завершение работы QEMU"
                    pkill -9 -f qemu-system-arm || true
                    echo "QEMU остановлен"
                    echo "Финальные логи QEMU"
                    cat qemu.log
                '''
                archiveArtifacts artifacts: 'qemu.log', fingerprint: true
            }
        }
    }
}
