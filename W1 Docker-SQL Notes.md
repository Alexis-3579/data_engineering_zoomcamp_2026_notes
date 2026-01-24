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

### Creating Virtual Environment 
In this course, we use uv to manage virtual environment. 
```
pip install uv

uv init --python 3.13 
```
- uv init initializes a new Python project with a standard file structure:
  - Creates pyproject.toml: The "brain" of your project where dependencies and metadata are listed.
  - Creates .python-version: A file that pins the specific version of Python you want to use.
  - Initializes Git: It automatically runs git init and creates a .gitignore tailored for Python.

To see the different versions of python used in the system and the virtual environment:
```
# Python in the virtual environment
uv run which python  
uv run python -V

# System Python
which python        
python -V
```
- Note: for uv run, if the virtual environment doesn't exist yet, uv run creates it automatically based on your pyproject.toml. And if you recently added a package (e.g., via uv add), uv run will install it into the environment before running your command.

- Next to tell VS code which python version we're using, type '>' in the search bar --> python select interpreter --> enter path (and choose the python inside .venv folder).
- To add dependencies
```
uv add pandas pyarrow
```
- Finally, to run the pipeline or your file
```
uv run file.py
```
  



### Creating Custom Docker Image 
**Docker file**: 
- Good article: https://www.geeksforgeeks.org/cloud-computing/what-is-dockerfile/
- A Dockerfile is a plain text document that contains all the instructions needed to automatically create a Docker image.
- It essentially is a blueprint for building a containerized application environment, defining a consistent and reproducible way to package an application with all its dependencies.
- <img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/13b66ead-9d8e-427e-a7bd-c5f5f81882a6" />

**Example:** 
```
# Dockerfile
FROM python:3.13.11-slim

# set up our image by installing prerequisites; pandas in this case
RUN pip install pandas pyarrow

# set up the working directory inside the container
WORKDIR /app
# copy the script to the container. 1st name is source file, 2nd is destination
COPY pipeline.py pipeline.py

# define what to do first when the container runs
# in this example, we will just run the script
ENTRYPOINT ["python", "pipeline.py"]

```
Explanation:
- FROM: Specifies the base image to start the build process
- RUN:  Executes a command during the image build process. This is often used for installing software packages, updating the system, or compiling applications.
- WORKDIR: Sets the working directory for any subsequent RUN, CMD, ENTRYPOINT, COPY, and ADD instructions in the docker container. 
- COPY <src> <dest>: Copies files or directories from the host machine to the container's filesystem at the specified destination.
- ENTRYPOINT: Configures a container that will run as an executable.
  - Default: If you don't use ENTRYPOINT, you have to tell Docker exactly what to do every time: docker run my-python-app(image name) python script.py --help (You are telling the container: "Run the python program, and give it these files.")
  - If you set ENTRYPOINT ["python", "script.py"] in your Dockerfile, the container becomes that script. You run it like this: docker run my-python-app --help
  - This pattern is used to package complex tools so they are easy for other people to use without knowing how they work or the interal file structure.
 
**What about uv?**

```
# Updated Dockerfile
FROM python:3.13.11-slim

WORKDIR /app

COPY pipeline.py pipeline.py

# Copy uv binary from official uv image (multi-stage build pattern)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/

#
COPY pyproject.toml .python-version uv.lock ./

# Tells uv to install dependencies exactly as they are defined in the uv lock file (the python environment set up previously)  
RUN uv sync --locked

RUN pip install pandas pyarrow

ENTRYPOINT ["uv", "run", "python", "pipeline.py"]

```
Another way of doing this if we don't want to include "uv" and "run" in the ENTRYPOINT is:
```
# Updated Dockerfile
FROM python:3.13.11-slim

WORKDIR /app

# Add virtual environment to PATH so we can use installed packages
ENV PATH="/code/.venv/bin:$PATH"

COPY pipeline.py pipeline.py

COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/

COPY pyproject.toml .python-version uv.lock ./

RUN uv sync --locked

RUN pip install pandas pyarrow

ENTRYPOINT ["python", "pipeline.py"]
```

**Build and Run the Image**

Finally, to build and run the image: 
```
docker build -t test:pandas .
```
```
docker run -it test:pandas 12 #random number
```



### Using Docker to Run Postgres 
To run PostgreSQL in a contianer: 

```
docker run -it --rm \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="ny_taxi" \
  -v ny_taxi_postgres_data:/var/lib/postgresql \
  -p 5432:5432 \
  postgres:18
```
**Explanation**
- `-e` (e stands for environment): sets environment variables (user, password, database name)
- `-v ny_taxi_postgres_data:/var/lib/postgresql` creates a named volume








