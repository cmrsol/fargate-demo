## Fargate Demo Application:
This repo contains a CloudFormation stack to build a simple Fargate stack. The
stack simply runs a container that is running a silly little web site. To use
for another purpose simply change the value for ```taskImage``` in the ```config.ini```
file. You can also give the container more CPU and/or memory by adjusting the
```cpuAllocation``` and/or ```memoryAllocation``` values in the same file. 

*Note the values for these parmameters must follow the following table:*

|cpuAllocation       |memoryAllocation                          |
|--------------------|------------------------------------------|
|0.25 vCPU           |0.5GB, 1GB, and 2GB                       |
|0.5 vCPU            |Min. 1GB and Max. 4GB, in 1GB increments  |
|1 vCPU              |Min. 2GB and Max. 8GB, in 1GB increments  |
|2 vCPU              |Min. 4GB and Max. 16GB, in 1GB increments |
|4 vCPU              |Min. 8GB and Max. 30GB, in 1GB increments |



## Build the Fargate Demo Stack:
```
# Make and activate a python virtual environment
virtualenv fargate-demo
. fargate-demo/bin/activate
pip install -Ur requirements.txt
stackility upsert -i config.ini
```
