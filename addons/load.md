cd /opt/dovechat/load

docker compose run --rm locust locust -f locustfile.py --host=http://localhost --users 100 --spawn-rate 10 --run-time 3m --headless --html=report_3_100.html
