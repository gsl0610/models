# Intel-Optimized TensorFlow Serving Installation (Linux)

## Goal
This tutorial will guide you through step-by-step instructions for
* [Installing Intel optimized TensorFlow Serving as Docker image](#installation)
* Running an example - [serving ResNet-50 v1 saved model using REST API and GRPC](#example-serving-resnet-50-v1-model).

## Prerequisites
1.  Access to a machine with the following configurations:
	* **Hardware recommendations**
		* minimum of **20 GB of free disk space** (required) 
		* minimum of **8 logical cores** (highly recommended).
		* We recommend the following Instance types for cloud VM's which have the latest Intel® Xeon® Processors:
			* AWS: [C5 Instances](https://aws.amazon.com/ec2/instance-types/c5/) (Helpful guides: [Get_Started](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html), [Accessing_Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html))
			* GCP: ["Intel Skylake" CPU Platform](https://cloud.google.com/compute/docs/cpu-platforms) (Helpful guides: [Get_Started](https://cloud.google.com/compute/docs/instances/create-start-instance), [Accessing_Instances](https://cloud.google.com/sdk/gcloud/reference/compute/ssh))
			* Azure: [Fsv2-series](https://azure.microsoft.com/en-us/pricing/details/virtual-machines/linux/#f-series) or [Hc-series](https://azure.microsoft.com/en-us/pricing/details/virtual-machines/linux/#hc-series) (Helpful guides: [Get_Started](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/), [Accessing_Instances](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-cli))

	* **Software recommendations**
		* Ubuntu 16.04 as these instructions were written and tested for it, but the process should be very similar for any other Linux distribution.
		* SSH login and HTTP/S traffic enabled. For details, contact your system administrator or see cloud provider console documentation ([AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html), [GCP](https://cloud.google.com/sdk/gcloud/reference/compute/ssh), [Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-cli)).

2. **Install Docker CE**
	*  Click [here](https://docs.docker.com/install/linux/docker-ce/ubuntu) for Ubuntu instructions. For other OS platforms, see [here](https://docs.docker.com/install/).
	* Setup docker to be used as a non-root user, to run `docker` commands without `sudo` . Exit and restart your SSH session so that your username is in effect in the docker group.
		```
		sudo usermod -aG docker `whoami`
		```
	* After exiting and restarting your SSH session, you should be able to run `docker` commands without `sudo`.
		```
		docker run hello-world
		```
		**NOTE**: If your machine is behind a proxy, See **HTTP/HTTPS proxy** section [here](https://docs.docker.com/config/daemon/systemd/)

## Installation
We will break down the installation into 2 steps: 
* Step 1: Build Intel Optimized TensorFlow Serving Docker image
* Step 2: Verify the Docker image by serving a simple model - half_plus_two

### Step 1: Build Intel Optimized TensorFlow Serving Docker image
The recommended way of using TensorFlow Serving is with Docker images. Lets build a docker image with Intel Optimized TensorFlow Serving. 

* Login into your machine via SSH and clone the [Tensorflow Serving](https://github.com/tensorflow/serving/) repository and save the path of this cloned directory (Also, adding it to `.bashrc` ) for ease of use for the remainder of this tutorial. 
	```
	$ git clone https://github.com/tensorflow/serving.git
	$ export TF_SERVING_ROOT=$(pwd)/serving
	$ echo "export TF_SERVING_ROOT=$(pwd)/serving" >> ~/.bashrc
	```
* Using `Dockerfile.devel.mkl`, build an image with Intel optimized ModelServer. This creates an image with all the required development tools and builds from sources. The image size will be around 5GB and will take some time. On AWS c5.4xlarge instance (16 logical cores), it took about 25min.
	```
	$ cd $TF_SERVING_ROOT/tensorflow_serving/tools/docker/
	$ docker build -f Dockerfile.devel-mkl -t tensorflow/serving:latest-devel-mkl .
	```
* Next, using `Dockerfile.mkl`, build a serving image which is a light-weight image without any development tools in it. `Dockerfile.mkl` will build a serving image by copying Intel optimized libraries and ModelServer from the development image built in the previous step - `tensorflow/serving:latest-devel-mkl `
	```
	$ cd $TF_SERVING_ROOT/tensorflow_serving/tools/docker/
	$ docker build -f Dockerfile.mkl -t tensorflow/serving:mkl .
	```

	**NOTE 1**: Docker build command require a `.` path argument at the end; see [docker examples](https://docs.docker.com/engine/reference/commandline/build/#examples) for more background.
		
	**NOTE 2**: If your machine is behind a proxy, you will need to pass proxy arguments to both build commands. For example:
	```
	--build-arg http_proxy="http://proxy.url:proxy_port" --build-arg https_proxy="http://proxy.url:proxy_port"
	```
* Once you built both the images, you should be able to list them using command `docker images`
	```
	$ docker images
	REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
	tensorflow/serving   mkl                 d33c8d849aa3        7 minutes ago       520MB
	tensorflow/serving   latest-devel-mkl    a2e69840d5cc        8 minutes ago       5.21GB
	ubuntu               18.04               20bb25d32758        13 days ago         87.5MB
	hello-world          latest              fce289e99eb9        5 weeks ago         1.84kB
	```
	
### Step 2: Verify the Docker image by serving a simple model - half_plus_two

Let us test the server by serving a simple mkl version of half_plus_two model which is included in the repo which we cloned in the previous step.

* Set the location of test model data:
	```
	$ export TEST_DATA=$TF_SERVING_ROOT/tensorflow_serving/servables/tensorflow/testdata
	```
* Start the container 
	* with `-p`, publish the container’s port 8501 to host's port 8501 where the TF serving listens to REST API requests
	* with `--name`, assign a name to the container for acessing later for checking status or killing it.
	* with `-v`,  mount the host local model directory `$TEST_DATA/saved_model_half_plus_two_mkl` on the container `/models/half_plus_two`.
	* with `-e`, setting an environment variable in the container which is read by TF serving
	* with `tensorflow/serving:mkl` docker image 
	* with `&` at the end, runs the container as a background process. Press enter after executing the following cmd:
	```
	$ docker run \
	  -p 8501:8501 \
	  --name tfserving_half_plus_two \
	  -v $TEST_DATA/saved_model_half_plus_two_mkl:/models/half_plus_two \
	  -e MODEL_NAME=half_plus_two \
	  tensorflow/serving:mkl &
	```

* Query the model using the predict API:
	```
	$ curl -d '{"instances": [1.0, 2.0, 5.0]}' \
	-X POST http://localhost:8501/v1/models/half_plus_two:predict
	```
	You should see the following output:
	```
	{
	"predictions": [2.5, 3.0, 4.5]
	}
	```
	NOTE: If you see any issues as below after sending predict request, please make sure to set your proxy (inside corporate environment)
	```
	$curl -d '{"instances": [1.0, 2.0, 5.0]}' \
		-X POST http://localhost:8501/v1/models/half_plus_two:predict \
		<http://localhost:8501/v1/models/half_plus_two:predict>
	<HTML>
	<HEAD><TITLE>Redirection</TITLE></HEAD>
	<BODY><H1>Redirect</H1></BODY>
	```
	Place this proxy information in your `~/.bashrc` or `/etc/environment`
	```
	export http_proxy="<http_proxy>"
	export https_proxy="<https_proxy>"
	export ftp_proxy="<ftp_proxy>"
	export socks_proxy="<socks_proxy>"
	export HTTP_PROXY=${http_proxy}
	export HTTPS_PROXY=${https_proxy}
	export FTP_PROXY=${ftp_proxy}
	export SOCKS_PROXY=${socks_proxy}
	export no_proxy=localhost,127.0.0.1,<add_your_machine_ip>,<add_your_machine_hostname>
	export NO_PROXY=${no_proxy}
	```

* After you are fininshed with querying, you can stop the container which is running in the background. To restart the container with the same name, you need to stop and remove the container from the registry. To view your running containers run `docker ps`.
	```
	$ docker rm -f tfserving_half_plus_two
	```

 *  **Note:** If you want to confirm that MKL optimizations are being used, add `-e MKLDNN_VERBOSE=1` to the `docker run` command.   This will log MKL messages in the docker logs, which you can inspect after a request is processed.
	```
	$ docker run \
	  -p 8501:8501 \
	  --name tfserving_half_plus_two \
	  -v $TEST_DATA/saved_model_half_plus_two_mkl:/models/half_plus_two \
	  -e MODEL_NAME=half_plus_two \
	  -e MKLDNN_VERBOSE=1 \
	  tensorflow/serving:mkl &
	```  
	 Query the model using the predict API as before:
    ```
    $ curl -d '{"instances": [1.0, 2.0, 5.0]}' \
    -X POST http://localhost:8501/v1/models/half_plus_two:predict
    ```
    You should see the result with MKLDNN verbose output like below:
    ```
    mkldnn_verbose,exec,reorder,simple:any,undef,in:f32_nhwc out:f32_nChw16c,num:1,1x1x10x10,0.00488281     
    mkldnn_verbose,exec,reorder,simple:any,undef,in:f32_hwio out:f32_OIhw16i16o,num:1,1x1x1x1,0.000976562
    mkldnn_verbose,exec,convolution,jit_1x1:avx512_common,forward_training,fsrc:nChw16c fwei:OIhw16i16o fbia:x fdst:nChw16c,alg:convolution_direct,mb1_g1ic1oc1_ih10oh10kh1sh1dh0ph0_iw10ow10kw1sw1dw0pw0,0.00805664
    mkldnn_verbose,exec,reorder,simple:any,undef,in:f32_nChw16c out:f32_nhwc,num:1,1x1x10x10,0.012207
    {
        "predictions": [2.5, 3.0, 4.5]
    }
    ```

## Example: Serving ResNet-50 v1 Model

TensorFlow Serving requires the model to be in SavedModel format. In this example, we will :
* Download a [pre-trained ResNet-50  v1 SavedModel](https://github.com/tensorflow/models/tree/master/official/resnet#pre-trained-model) and 
* use the [python client code](https://github.com/tensorflow/serving/tree/master/tensorflow_serving/example) from the TensorFlow Serving repository and query using two methods:
	* [Using REST API](#option-1-query-using-rest-api), which is simple to set up, but lacks performance when compared with GRPC
	* [Using GRPC](#option-2-query-using-grpc), which has optimal performance but the client code requires additional dependencies to be installed.

**NOTE:** NCHW data format is optimal for Intel-optimized TensorFlow Serving.

#### Download and untar a ResNet-50 v1 SavedModel to `/tmp/resnet`
```
$ mkdir /tmp/resnet  
$ curl -s http://download.tensorflow.org/models/official/20181001_resnet/savedmodels/resnet_v1_fp32_savedmodel_NCHW_jpg.tar.gz \
| tar --strip-components=2 -C /tmp/resnet -xvz
```

### Option 1: Query using REST API 
* Querying using REST API is simple to set up, but lacks performance when compared with GRPC.

* If a running container is using port 8501, you need to stop it. View your running containers with `docker ps`.  To stop and remove the contatiner from the registry, copy the `CONTAINER ID` from the  `docker ps` output and run `docker rm -f <container_id>`.

* Start the container 
	* with `-p`, publish the container’s port 8501 to host's port 8501 where the TF serving listens to REST API requests
	* with `--name`, assign a name to the container for acessing later for checking status or killing it.
	* with `-v`,  mount the host local model directory `/tmp/resnet` on the container `/models/resnet`.
	* with `-e`, setting an environment variable in the container which is read by TF serving
	* with `tensorflow/serving:mkl` docker image 
	* with `&` at the end, runs the container as a background process. Press enter after executing the following cmd:
	```
	 $ docker run \
	 -p 8501:8501 \
	 --name=tfserving_resnet_restapi \
	 -v "/tmp/resnet:/models/resnet" \
	 -e MODEL_NAME=resnet \
	 tensorflow/serving:mkl &
	```
* Install the prerequisites for running the python client code
	```
	$ sudo apt-get install -y python python-requests
	```
* Run the example `resnet_client.py` script from the TensorFlow Serving repository
	```
	$ python $TF_SERVING_ROOT/tensorflow_serving/example/resnet_client.py
	```
	You should see the following output:
	```
	Prediction class: 286, avg latency: 34.7315 ms
	```
  **Note:** The real avg latency you see will depend on your hardware, environment, and whether or not you have configured the server parameters optimally. See the [General Best Practices](GeneralBestPractices.md) for more information.
  
* After you are fininshed with querying, you can stop the container which is running in the background. To restart the container with the same name, you need to stop and remove the container from the registry. To view your running containers run `docker ps`. 
	```
	$ docker rm -f tfserving_resnet_restapi
	```

### Option 2: Query using GRPC 
* Querying using GRPC will have optimal performance but the client code requires additional dependencies to be installed.

* If a running container is using port 8500, you need to stop it. View your running containers with `docker ps`.  To stop and remove the contatiner from the registry, copy the `CONTAINER ID` from the  `docker ps` output and run `docker rm -f <container_id>`.

* Start a container 
	* with `-p`, publish the container’s port 8500 to host's port 8500 where the TF serving listens to GRPC requests
	* with `--name`, assign a name to the container for acessing later for checking status or killing it.
	* with `-v`,  mount the host local model directory `/tmp/resnet` on the container `/models/resnet`.
	* with `-e`, setting an environment variable in the container which is read by TF serving
	* with `tensorflow/serving:mkl` docker image 
 	* with `&` at the end, runs the container as a background process. Press enter after executing the following cmd:
	```
	 $ docker run \
	 -p 8500:8500 \
	 --name=tfserving_resnet_grpc \
	 -v "/tmp/resnet:/models/resnet" \
	 -e MODEL_NAME=resnet \
	 tensorflow/serving:mkl &
	```
* You will need a few python packages in order to run the client, we recommend installing them in a virtual environment. 
	```
	$ sudo apt-get install -y python python-pip
	$ pip install virtualenv
	```
* Create and activate the python virtual envirnoment. Install the packages needed for the GRPC client.
	```
	$ cd ~
	$ virtualenv tfserving_venv
	$ source tfserving_venv/bin/activate
	(tfserving_venv)$ pip install grpc requests tensorflow tensorflow-serving-api
	```
* Run the example `resnet_client_grpc.py` script from the TensorFlow Serving repository, which you cloned earlier.
	```
	(tfserving_venv)$ python $TF_SERVING_ROOT/tensorflow_serving/example/resnet_client_grpc.py
	```
  You should see the similar output as below:
	```
	outputs {
	  key: "classes"
	  value {
	    dtype: DT_INT64
	    tensor_shape {
	      dim {
	        size: 1
	      }
	    }
	    int64_val: 286
	  }
	}
	outputs {
	  key: "probabilities"
	  value {
	    dtype: DT_FLOAT
	    tensor_shape {
	      dim {
	        size: 1
	      }
	      dim {
	        size: 1001
	      }
	    }
	    float_val: 7.8115895974e-08
	    float_val: 3.93756813821e-08
	    float_val: 6.0871172991e-07
	  .....
	  .....
	  }
	}
	model_spec {
	  name: "resnet"
	  version {
	    value: 1538686758
	  }
	  signature_name: "serving_default"
	}
	```
  
 
* To deactivate your virtual environment:
	```
	(tensorflow-serving-api)$ deactivate
	```

* After you are fininshed with querying, you can stop the container which is running in the background. To restart the container with the same name, you need to stop and remove the container from the registry. To view your running containers run `docker ps`. 
	```
	$ docker rm -f tfserving_resnet_grpc
	```


## Debugging

If you have any problems while making a request, the best way to debug is to check the docker logs.
First, find the Container ID of your running docker container with `docker ps` and then view its logs with `docker logs <container_id>`.
If you have added `-e MKLDNN_VERBOSE=1` to the `docker run` command, you should see mkldnn_verbose messages too.

