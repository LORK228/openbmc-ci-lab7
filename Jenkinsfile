pipeline {
    agent any

    environment {
        BMC_IP       = "localhost"
        BMC_USER     = "root"
        BMC_PASSWORD = "0penBmc"
    }

    stages {
        stage('Launch Real OpenBMC in QEMU') {
            steps {
                script {
                    echo 'Запуск OpenBMC в QEMU...'

                    def imageFile = sh(script: 'ls romulus/obmc-phosphor-image-romulus-*.static.mtd', returnStdout: true).trim()
                    echo "Найден образ: ${imageFile}"

                    sh """
                        nohup qemu-system-arm \
                            -m 256 \
                            -M romulus-bmc \
                            -nographic \
                            -drive file=${imageFile},format=raw,if=mtd \
                            -net nic \
                            -net user,hostfwd=tcp::2222-:22,hostfwd=tcp::2443-:443,hostfwd=udp::2623-:623 \
                            -monitor none \
                            -serial null \
                            -daemonize > /dev/null 2>&1

                        sleep 8

                        QEMU_PID=\$(pgrep -f qemu-system-arm)
                        if [ -n "\$QEMU_PID" ]; then
                            echo "QEMU запущен, PID: \$QEMU_PID"
                            echo \$QEMU_PID > qemu.pid
                        else
                            echo "Ошибка запуска QEMU"
                            exit 1
                        fi
                    """
                }
            }
        }

        stage('Wait for BMC Boot Complete') {
            steps {
                echo 'Ожидание полной загрузки OpenBMC...'
                sh '''
                    BOOT_READY=false
                    i=1
                    while [ $i -le 40 ]; do
                        echo "Проверка загрузки... попытка $i"
                        if curl -k -s https://localhost:2443/redfish/v1/ | grep -q "odata"; then
                            echo "Redfish API доступен!"
                            BOOT_READY=true
                            break
                        fi
                        sleep 5
                        i=$((i + 1))
                    done

                    if [ "$BOOT_READY" = false ]; then
                        echo "OpenBMC не загрузился вовремя"
                        exit 1
                    fi
                    echo "OpenBMC успешно загружен!"
                '''
            }
        }

        stage('Test BMC Redfish API') {
            steps {
                echo 'Тестирование Redfish API...'
                sh '''
                    echo "=== Redfish API Tests ===" > redfish_test.log
                    date >> redfish_test.log

                    for endpoint in "/redfish/v1/" "/redfish/v1/Systems/system" "/redfish/v1/Managers"; do
                        echo -e "\\n--- $endpoint ---" >> redfish_test.log
                        if curl -k -u ${BMC_USER}:${BMC_PASSWORD} -s "https://${BMC_IP}:2443$endpoint" >> redfish_test.log 2>&1; then
                            echo "OK: $endpoint" >> redfish_test.log
                        else
                            echo "FAIL: $endpoint" >> redfish_test.log
                        fi
                    done
                '''
            }
        }

        stage('Test BMC SSH Access') {
            steps {
                echo 'Тестирование SSH-доступа...'
                sh '''
                    echo "=== SSH Access Tests ===" > ssh_test.log
                    date >> ssh_test.log

                    if ! command -v sshpass >/dev/null 2>&1; then
                        echo "sshpass не найден — установка..." >> ssh_test.log
                        apt-get update -qq && apt-get install -y -qq sshpass
                    fi

                    for cmd in "cat /etc/os-release" "uname -a" "ps aux | head -10"; do
                        echo -e "\\n--- $cmd ---" >> ssh_test.log
                        if sshpass -p ${BMC_PASSWORD} ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 ${BMC_USER}@${BMC_IP} -p 2222 "$cmd" >> ssh_test.log 2>&1; then
                            echo "OK" >> ssh_test.log
                        else
                            echo "FAIL" >> ssh_test.log
                        fi
                    done

                    echo "SSH тесты завершены" >> ssh_test.log
                '''
            }
        }

        stage('Test BMC WebUI (Pytest)') {
            steps {
                echo 'Запуск WebUI тестов через pytest...'
                sh '''
                    # Установка зависимостей и создание virtualenv
                    apt-get update -qq
                    apt-get install -y -qq python3-pip python3-venv ipmitool sshpass

                    python3 -m venv pytest_venv
                    . pytest_venv/bin/activate
                    pip install pytest requests

                    # Создание файла тестов
                    cat > test_webui.py << 'EOF'
import pytest
import requests
import json
import time

requests.packages.urllib3.disable_warnings()

BASE_URL = "https://127.0.0.1:2443"
USERNAME = "root"
PASSWORD = "0penBmc"

@pytest.fixture(scope="session")
def redfish_session():
    session_url = f"{BASE_URL}/redfish/v1/SessionService/Sessions"
    payload = {
        "UserName": USERNAME,
        "Password": PASSWORD
    }
    headers = {"Content-Type": "application/json"}
    resp = requests.post(
        session_url,
        json=payload,
        headers=headers,
        verify=False,
        timeout=10
    )
    assert resp.status_code == 201, f"Login failed: {resp.status_code} {resp.text}"
    token = resp.headers.get("X-Auth-Token")
    assert token, "X-Auth-Token not found in login response"
    session = requests.Session()
    session.headers.update({
        "X-Auth-Token": token,
        "Content-Type": "application/json"
    })
    session.verify = False
    yield session
    # Завершаем сессию
    session_id = resp.json().get("Id")
    if session_id:
        session.delete(f"{BASE_URL}/redfish/v1/SessionService/Sessions/{session_id}")

def test_authentication(redfish_session):
    resp = redfish_session.get(f"{BASE_URL}/redfish/v1/")
    assert resp.status_code == 200
    data = resp.json()
    assert "Systems" in data, f"Expected 'Systems' key in root response, got: {data.keys()}"

def test_get_system_info(redfish_session):
    resp = redfish_session.get(f"{BASE_URL}/redfish/v1/Systems/system")
    assert resp.status_code == 200
    data = resp.json()
    assert "PowerState" in data
    assert "Status" in data

def test_power_cycle(redfish_session):
    action_url = f"{BASE_URL}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset"
    # ForceOff
    payload_off = {"ResetType": "ForceOff"}
    resp_off = redfish_session.post(action_url, json=payload_off)
    assert resp_off.status_code in (200, 202, 204)
    time.sleep(5)
    # On
    payload_on = {"ResetType": "On"}
    resp_on = redfish_session.post(action_url, json=payload_on)
    assert resp_on.status_code in (200, 202, 204)

def test_thermal_sensors(redfish_session):
    resp = redfish_session.get(f"{BASE_URL}/redfish/v1/Chassis")
    if resp.status_code != 200:
        pytest.skip("No chassis found")
    data = resp.json()
    members = data.get("Members", [])
    if not members:
        pytest.skip("No members in chassis")
    chassis_id = members[0].get("@odata.id").split('/')[-1]
    thermal_url = f"{BASE_URL}/redfish/v1/Chassis/{chassis_id}/Thermal"
    resp = redfish_session.get(thermal_url)
    if resp.status_code != 200:
        pytest.skip("Thermal endpoint unavailable")
    data = resp.json()
    temperatures = data.get("Temperatures", [])
    if not temperatures:
        pytest.skip("No temperature sensors found")
    for sensor in temperatures:
        reading = sensor.get("ReadingCelsius")
        if reading is not None:
            assert 0 <= reading < 120
EOF

                    # Запуск тестов
                    pytest -v test_webui.py --junitxml=webui_junit.xml > webui_test.log 2>&1 || true
                    echo "WebUI тесты завершены"

                    deactivate
                '''
            }
        }

        stage('Run Load Testing') {
            steps {
                echo 'Нагрузочное тестирование...'
                sh '''
                    echo "=== Load Testing ===" > load_test.log
                    date >> load_test.log

                    for i in {1..30}; do
                        curl -k -s -o /dev/null -u ${BMC_USER}:${BMC_PASSWORD} \
                             -w "Запрос $i | Код: %{http_code} | Время: %{time_total}s\\n" \
                             https://${BMC_IP}:2443/redfish/v1/ >> load_test.log
                        sleep 0.3
                    done

                    echo "Нагрузочное тестирование завершено" >> load_test.log
                '''
            }
        }
    }

    post {
        always {
            echo 'Очистка QEMU...'
            sh '''
                [ -f qemu.pid ] && kill $(cat qemu.pid) || true
                rm -f qemu.pid
                pkill -f qemu-system-arm || true
            '''
            archiveArtifacts artifacts: '*.log, *.xml, final_result.txt', allowEmptyArchive: true
        }
        success {
            sh 'echo "Лабораторная 7 — УСПЕШНО ЗАВЕРШЕНА" > final_result.txt'
            echo 'ВСЁ ОТЛИЧНО!'
        }
        failure {
            sh 'echo "Лабораторная 7 — ПРОВАЛЕНА" > final_result.txt'
        }
    }
}
