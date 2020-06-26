# genea_visualizer

The server consists of several containers which are launched together with the docker-compose command described below.
The componantes are:
* web: this is the http server which receives render requests and places them on a "celery" queue to be processed.
* worker: this takes jobs from the "celery" queue and works on them. Each worker runs one blender process, so increasing the amount of workers adds more parallelization. 
* monitor: this is a monitoring tool for celery. Default username is `***REMOVED***` and password is `***REMOVED***` (can be changed by setting `FLOWER_USER` and `FLOWER_PWD` when starting the docker-compose command)
* redis: needed for celery

First you need to install docker-compose:
`sudo apt  install docker-compose` (on ubuntu)

Then to start the server run `docker-compose up --build`

In order to run several (for example 3) workers (blender renderers, which allows to parallelize rendering) `docker-compose up --build --scale worker=3`

The `-d` flag can also be passed in order to run the server in the background. Logs can then be accessed by running `docker-compose logs -f`. Additionally it's possible to rebuild just the worker or api containers with minimal distruption in the running server by running for example `docker-compose up -d --no-deps --scale worker=2 --build worker`. This will rebuild the worker container and stop the old ones and start 2 new ones.


The server is http based and works by uploading a bvh file. You will then recieve a "job id" which you can poll in order to see the progress of your rendering. When it is finished you will receive a URL to a video file that you can download. 
Below are some examples using `curl` and at the bottom of the page (and in the file `example.py`) is a full python example of how this can be used.

Since the server is avialable publicly online, we have a simple authentication system, so just pass in the token `j7HgTkwt24yKWfHPpFG3eoydJK6syAsz` with each request.

```curl -XPOST -H "Authorization:Bearer j7HgTkwt24yKWfHPpFG3eoydJK6syAsz" -F "file=@/path/to/bvh/file.bvh" http://SERVER_URL/render``` 
will return a URI to the current job `/jobid/[JOB_ID]`.

`curl -H "Authorization:Bearer j7HgTkwt24yKWfHPpFG3eoydJK6syAsz" http://SERVER_URL/jobid/[JOB_ID]` will return the current job state, which might be

* `{result": {"jobs_in_queue": X}, "state": "PENDING"}`: Which means the job is in the queue and waiting to be rendered. The `jobs_in_queue` property is the total number of jobs waiting to be executed. The order of job execution is not guranteed, which means that this number does not reflect how many jobs there are before the current job, but rather reflects if the server is currently busy or not.

* `{result": null, "state": "PROCESSING"}`: The job is currently being processed. Depending on the file size this might take a while, but this acknowledges that the server has started to working on the request.

* `{result":{"current": X, "total": Y}, "state": "RENDERING"}`: The job is currently being rendered, this is the last stage of the process. `current` shows which is the last rendered frame and `total` shows how many frames in total this job will render.

* `{"result": FILE_URL, "state": "SUCCESS"}`: The job ended successfully and the video is available at `http://SERVER_URL/[FILE_URL]`.

* `{"result": ERROR_MSG, "state": "FAILURE"}`: The job ended with a failure and the error message is given in `results`.


In order to retrieve the video: `curl -H "Authorization:Bearer j7HgTkwt24yKWfHPpFG3eoydJK6syAsz" http://SERVER_URL/[FILE_URL] -o result.mp4`. Please note that the server will delete the file after you retrieve it, so you can only retrieve it once!


A full python (3.7) example can also be found in `example.py`.