To replace the default long prompt with a >, we can type ` echo 'PS1="> "' > ~/.bashrc ` in the terminal. 


### What is Docker? 
- Good article:  https://www.geeksforgeeks.org/devops/introduction-to-docker/
Motivation for docker: before docker, deploying applications across environments was difficult, often encountering 'it works on my machine' problems due to dependencies, configs and OS-variations. 
How docker solves this problem: 
- Docker standardizes the runtime environment by bundling everything (app + dependencies) into a single, immutable container.
- Docker is an OS-level virtualization. Different to VMs that virutalize hardware and runs multiple guest OSs, Docker shares the host kernel and only isolates the specific software stack required to run the application.


### What is Docker Image? 
- A docker image is a read-only **template** that contains the OS, your code and all the libraries needed to run the application.
- It is a immutable snapshot that never changes once it's built. 
- It is stored on disk.

### What is Docker Container?
- A docker container is like an instance of the image.
- We can 100 containers from the same image, they all share the exact same physical image files on your hard drive. They don't take up 100x the space. Only the tiny "writable layer" is unique to each container.

### Containers are Stateless 
- Docker containers are stateless (i.e. it doesn't preserve state). The container itself doesn't remember anything that happened to it while it was running.
- Every time we start a container from an image, it begins in a clean, identical state.
- A Docker image is a 'read-only' template layer that contains the OS, Python and your code; when you start a containter, Docker adds a thin writeable layer on top (this is where the new files, logs, and database entries are stored.
- Once the container is delete, the container layer and all its data is gone.
- Caveat: although the files are not gone when the container is stopped, relying on stopped containers to preserve state is a bad practice.
- Why?
  1. If we accidentally deletes the container, the writeable layer is gone instantly.
  2. Inefficient: Docker uses a 'copy-on-write' system. Writing large amounts of data (like a database) directly to a container's layer is significantly slower than writing to a native volume.
  3. Low portability: Data stuck inside a stopped container is trapped. You can’t easily move that data to a newer version of the image or share it with another container.


### Getting Started with Docker - Basic Commands 
- To see if docker is working properly, we can run `docker run hello-world` for a quick test. 
-  To have a bash session here instead of python (inside the python image), override the entrypoint: `docker run -it entrypoint=bash python:3.13.11-slim`.
-  To see (-a for showing everything; -aq to only show the ids) 
```
docker ps -a
```
- To delete the stopped containers
```
docker rm $(docker ps -aq)
```

### Preserving State in Docker -- Volume Mapping 
- Recall that, docker containers are stateless. The solution to this is volume mapping.
- Volume mapping is a process that links a directory or file on the host machine to a directory inside a container, ensuring data persistence and sharing.
- It provides bi-directional synchronization between the host and the container: any changes made in the container is immediately reflected on the host, and any changes made on the host are also reflected in the container when the changes are saved on the host.
- Syntax (using the -v flag):
```
docker run -v <source>:<destination>[:<options>] [OPTIONS] IMAGE [COMMAND]
```
- Example:
```
docker run -it -v $(pwd)/test:/app/test --entrypoint=bash python:3.13.11-slim
```
Note: if the above produces '\Program Files\Git\app\test', it is likely that Git Bash Path Conversion has occurred. In this case, use double quotes and double slash. 
```
docker run -it -v "/$(pwd)/test://app/test" --entrypoint=bash python:3.13.11-slim
```

### Creating Custom Docker Image 
- Docker file 

