## Prerequisites
- Install docker engine and docker compose in the linux system (Ubuntu 20.04)
```sh
sudo apt-get update -y
sudo apt-get install ca-certificates curl gnupg lsb-release -y
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```

- Check the status of docker and add the current user in docker group.
```sh
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
sudo docker version
sudo docker compose version
sudo usermod -aG docker $(users)
bash --login
```

- Pull the below mentioned images in the system.
```sh
docker pull infracloudio/csvserver:latest
docker pull prom/prometheus:v2.22.0
```
	
## Part I
1. Run the container with image `infracloudio/csvserver:latest` in background and check if it's running.
```sh
docker run --name csvserver -it -d infracloudio/csvserver:latest 
docker container logs csvserver
```

2. Error Found:- 
> **NOTE**: ERROR:- 2022/09/20 05:23:53 error while reading the file "/csvserver/inputdata": open /csvserver/inputdata: no such file or directory

3. Write the bash script `gencsv.sh` to generate a file `inputFile`
```sh
vim gencsv.sh
#!/bin/bash

# Blank the file before starting the script.
cat /dev/null > inputFile

# Create a blank file in the current directory.
FNAME=inputFile
touch ${FNAME}

# Now generate some random number with index and append it into above created file.
RANDOM=$(shuf -i 1-100 -n1)
NUM=0
ARG=${1:-10}

# Now generate some random number with index and append it into above created file.
if ! [[ "$ARG" =~ ^-?[0-9]+$ ]] || [[ ${ARG} -lt 0 ]]; then
	echo -e "Argument can't be in minus value and can't be string value!"
	exit 1
fi
while [[ ${NUM} -lt ${ARG} ]]
do
	echo -e "${NUM}, ${RANDOM}" >> ${FNAME}
	((NUM = NUM + 1))
done

# Change the permission of file so that all user can read the file.
chmod 664 ${FNAME}
```
save the `gencsv.sh` file and give the executable permission.
```sh
chmod +x gencsv.sh
```
- If script is executed without any argument then 10 entries will be append in `inputFile`
- If script is executed with argument as negative value or string then error will be display.
- if script is executed any valid integer value then same entries will be append in `inputFile`
- `inputFile` is readable to all usres.

4. Run the container again and copy the `inputFile` as `inputdata` file name in /csvserver path
```sh
docker container rm -f csvserver
docker run --name csvserver -it -d infracloudio/csvserver:latest
docker cp inputFile csvserver:/csvserver/inputdata
docker container start csvserver
docker container logs csvserver
```
	
5. Get shell access of the container and get the port on which application is listening.
```sh
docker container exec -it csvserver bash
netstat -an | grep -i LISTEN | awk '{print $4}'
```

6. Run the container again, same as step 4 and make sure that `inputFile` is copied in the container and application should be accessible at http://localhost:9393
```sh
docker container rm -f csvserver
docker run --name csvserver -it -d -p 9393:9300 infracloudio/csvserver:latest 
docker cp inputFile csvserver:/csvserver/inputdata
docker container start csvserver
docker container logs csvserver
```

- Access the application from browser at http://localhost:9393
- Set the environment variable in the container `CSVSERVER_BORDER` to value `Orange`
```sh
docker container exec -it csvserver bash
echo "export CSVSERVER_BORDER=Orange" >> $HOME/.bashrc
```
