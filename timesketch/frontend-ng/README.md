# Timesketch web frontend

## Project setup
```
yarn install
```

### Compiles and hot-reloads for development
```
yarn run serve
```

### Compiles and minifies for production
```
yarn run build
```

### Run your tests
```
yarn run test
```

### Lints and fixes files
```
yarn run lint
```

### Run your unit tests
```
yarn run test:unit
```

## Run the setup using Docker

```
git clone timesketch # replace with SSH or HTTP link
cd timesketch/docker/dev
docker-compose up -d # build the containers (-d to run them in the background)
docker-compose exec timesketch tsctl create-user dev --password dev # create user (dev, dev)
CONTAINER_ID="$(docker container list -f name=timesketch-dev -q)"
docker exec -d $CONTAINER_ID celery -A timesketch.lib.tasks worker --loglevel info

--- Frontend-ng development ---

docker-compose exec timesketch yarn install --cwd=/usr/local/src/timesketch/timesketch/frontend-ng
docker-compose exec timesketch yarn run --cwd=/usr/local/src/timesketch/timesketch/frontend-ng build --mode development --watch

# The previous command will "block" your shell, thus, you need to open another one (same folder).

CONTAINER_ID="$(docker container list -f name=timesketch-dev -q)"
docker exec -it $CONTAINER_ID /bin/bash # get the shell following command must be run inside the container

cd /usr/local/src/timesketch/timesketch/frontend-ng
npm install 
yarn install

# In your timesketch docker container, edit /etc/timesketch/timesketch.conf and set WTF_CSRF_ENABLED = False

exit # exit from the container

docker exec -it $CONTAINER_ID gunicorn --reload -b 0.0.0.0:5000 --log-file - --timeout 600 -c /usr/local/src/timesketch/data/gunicorn_config.py timesketch.wsgi:application 

# The previous command will block your shell, thus, you need to open another one (same folder).

CONTAINER_ID="$(docker container list -f name=timesketch-dev -q)"
docker-compose exec timesketch yarn run --cwd=/usr/local/src/timesketch/timesketch/frontend-ng serve
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).