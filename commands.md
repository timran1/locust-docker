```
sudo docker build  --no-cache -t locust ./
sudo docker tag locust timran1/locust:1.0
sudo docker push timran1/locust:1.0

docker run -it --rm locust

sudo docker image rm timran1/locust:1.0 --force
```
