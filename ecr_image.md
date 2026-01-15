# How to push trainer image onto server team's Amazon ECR repository

## Create a temp container with installed aws cli and docker
```bash
docker run --rm -it --name aws_ecr_tmp -v /var/run/docker.sock:/var/run/docker.sock aws_ecr:latest /bin/bash
```
## In container
```bash
# check if the images mounted successfully
docker images

# set aws config if needed. the "AWS Access Key ID", "AWS Secret Access Key", "Default region name" are given by server team
aws configure

# login aws ecr
# the server "073012653937.dkr.ecr.us-west-2.amazonaws.com" is defined by server team 成功會出現 Login Succeeded
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 690946208568.dkr.ecr.us-west-2.amazonaws.com

# docker pull images (ex: pull ginza six image)
docker pull 690946208568.dkr.ecr.us-west-2.amazonaws.com/yce/ai/apt-22:ginza_six_20260101

# create tag with tag name specified by server team for pushing
# for test: ai:avatar_demo  
docker tag avatar/trainer:20230222 073012653937.dkr.ecr.us-west-2.amazonaws.com/yce/ai:avatar_demo
# for prod: ai:avatar_us
docker tag avatar/trainer:20230222 073012653937.dkr.ecr.us-west-2.amazonaws.com/yce/ai:avatar_us

# push image (may take about 20~30 minutes if all layers need to be updated)
# for test: ai:avatar_demo  
docker push 073012653937.dkr.ecr.us-west-2.amazonaws.com/yce/ai:avatar_demo
# for prod: ai:avatar_us
docker push 073012653937.dkr.ecr.us-west-2.amazonaws.com/yce/ai:avatar_us
```
