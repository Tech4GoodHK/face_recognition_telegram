# Face Recognition Telegram Bot
## Index
1. [Installation for deployment](#Set-up)
2. [Understanding the structure of this project](#Structrue-of-the-project)
3. [Installation for development](#Prepare-pre-commit-hook)

# Prepare pre-commit hook
```
# Run this only if you are developing this bot
chmod +x prepare.sh
./prepare.sh
# you may have to comment out line 8-9 of prepare.sh if you didn't download this file via 'git clone'
# You may also have to manually install the modules in *requirements.txt as instructed by the error messages.
```
# Set up
## Credentials
Create the following four files such that the name 

Please read `https://core.telegram.org/bots#6-botfather` for instruction to generate bot token. <br/>
*** Use `/setprivacy` to disable privacy, if bot is used in group ***
- Development bot token `./dev_token.txt` (the token that botfather sends to you after creating the bot with the line `Use this token to access the HTTP API:`)
    - `123456789:10+_LETTERS_long_ALPHA_numeric_hash`
- Development root user token `./dev_root_token.txt`
    - `@name_of_bot`
- Production bot token `./prod_token.txt` (the token that botfather sends to you after creating the bot with the line `Use this token to access the HTTP API:`)
    - `123456789:10+_LETTERS_long_ALPHA_numeric_hash`
- Production root user token `./prod_root_token.txt`
    - `@name_of_bot`

## Allowing others to use the bot when in production mode
Edit the configuration in model/prod_config.py to the following
```
PUBLIC_USER = True
```
such that everyone can use the bot for prediction.

## Credentials for test case

## Docker instructions
It's most ideal to use docker for managing this bot. <br/>
Please remember to re-build the container after any changes.

You may have to run the following as root.
```
# Build container for development
docker-compose build
# Run bot for development
docker-compose run recog
# Run bot (detached) for development
docker-compose up -d
# Stop bot for development
docker-compose down
# Read logs
docker-compose logs recog

# Build container for production
docker-compose -f docker-compose-prod.yml build
# Run bot for production
docker-compose -f docker-compose-prod.yml run recog
# Run bot (detached) for production
docker-compose -f docker-compose-prod.yml up -d
# Stop bot for production
docker-compose -f docker-compose-prod.yml down
# Read logs
docker-compose -f docker-compose-prod.yml logs recog
```
##Testing
The following credentials are required to run test cases on this bot. Please read `https://core.telegram.org/api/obtaining_api_id` for instruction to generate API credentials.
- Telegram account session `./test/user.session`
- User credentials `./test/user_token.txt`
    - API ID
    - API HASH
    - BOT's id
    - USER's id <br/>
        - Example:
            - 123456
            - a1234567
            - @face_recognition_1234_bot
            - @user123456
Use the script `create_session.py` with the above API credentials to generate the session file.

# Build container for testing
docker-compose -f docker-compose-test.yml build
# Run bot for testing
docker-compose -f docker-compose-test.yml run recog
```
If stuck, see the [section](#Docker) for help/debugging.

## check number of requests
In your commandline, as root user,
```
docker-compose exec mongo mongo
```
which should connect you to a Mongo Shell at `mongodb://127.0.0.1:27017/`
```
> use default
> db.counters.find().pretty()
```

# Deploying into cloud
## DigitalOcean
Create account at Digital Ocean, choose the default plan (4GB, 2cpu). (The cheapest plan (1GB, 1CPU) does not have enough memory for building dlib)

Copy your ssh key into the Droplet's IP address:
```
FACE=xxx.xx.xxx.xx
ssh-copy-id root@$FACE
```

## use docker-machine
Install the latest version of docker machine (you may have to do ```sudo apt-get update; sudo apt-get upgrade```) and then
```
docker-machine create --driver generic --generic-ip-address=$FACE --generic-ssh-key <absolute path to your home directory>/.ssh/id_rsa --generic-ssh-user root vm  
#if the line above doesn't work, remove the --generic-ssh-key <absolute path to your home directory>/.ssh/id_rsa and try again
#use docker-machine ls to check that "vm" has been created.
```
After that, 
```
eval $(docker-machine env vm)
```
will make your docker run on the Droplet instead of the local machine.

At this point, re-running the [Docker instructions above](#Docker-instructions) will do the job.

# Structrue of the project
This repository contains the scripts necessary to host a facial-recognition bot on Telegram.

The picture sent to the bot will pass through a [DNN](#Neural-Network), which outputs a vector of len=128 for each person in the picture.

This vector (called a face\_encoding) is then compared against the existing face\_encoding's in the database to find a matching person. A KNN is use used to do this.

The name of this person/ these people is then returned to the user of the bot in a reply message.

This whole process is done from inside a [docker](#Docker).

In order to run it 24/7 you are advised to host it on Cloud (e.g. using service such as DigitalOcean).

## Neural Network
The controller/client.py imports the [face_recognition module](https://github.com/ageitgey/face_recognition) for face recognition:
face\_recognition uses dlib's [default model](https://github.com/ageitgey/face_recognition_models/)
which is a (29 layer) model, obtained from [here](https://github.com/davisking/dlib-models#dlib-models)

## Possible future improvements to the DNN
The `metric_network_resnet-0627.dat` file saves a model (containing the weights and biases) of the neural network.

If one follows and reads the links in the [section above](#Neural-Network), they can find out how to create a similar file. (Which may or may not involve replicating the [ResNet-34 network from the paper described here](https://github.com/davisking/dlib-models#dlib-models)). `metric_network_resnet-0627.dat` in particular is created with the same structure as dlib's default model; but is trained with 2M out of 6M being Asian faces.

If you wone to re-create this, below are some databases which may be useful for this purpose.
- `http://dlib.net/dnn_metric_learning_on_images_ex.cpp.html`
- `https://github.com/deepinsight/insightface/wiki/Dataset-Zoo`
- `https://drive.google.com/drive/folders/1ADcZugpo8Z6o5q1p2tIAibwhsL8DcVwH`
- `https://github.com/deepinsight/insightface/issues/256`

Once a new .dat file is generated and saved in the top directory, change the MODEL\_NAME in model/*\_config.py to your\_new\_model.dat, re-build docker, and then send the "/retrain" command to the bot as root\_admin/admin, so new face_encodings are created instead.

##Possible future improvements to the KNN
The KNN may be improved by using an SVM in its place; but I suspect that the improvement is limited unless the neural network is already trained to better pick out differences across Asian faces.

##Possible future improvements
However, the limiting factor to this project is the number of police photo that is obtained. Therefore, currently, the need to acquire data is much more pressing than the concerns in the two sections above.

##Docker
If you have never used docker before: it is like a virutal machine that runs on your computer.

After installing docker, make sure that you're using the [latest community version](https://docs.docker.com/install/linux/docker-ce/debian/)
```
docker -v
# This should give an output of version 1.24.1 (or later)
```
dlib compiling will take a significant amount of time; but because docker has incremental build it will only be done once on your machine.

Remember to re-build docker after any changes to files, especially after the [*token.txt files](#Credentials)

## Permissions
The following table summarizes who can interact with the bot in what configuration:

| Configuration	| Production	| Development	| Testing	|
|:------------- |:-------------:|:-------------:|:-------------:|
| user		|	Y	|	N	|	N	|
| admin		|	Y	|	N	|	N	|
| root_admin	|	Y	|	Y	|	Y	|

(This requires the step in the [section above](#Allowing-others-to-use-the-bot-when-in-production-mode) to be carried out)

The commands that user, admin, and root\_admin can do on the bot are listed in `user_model.py`
