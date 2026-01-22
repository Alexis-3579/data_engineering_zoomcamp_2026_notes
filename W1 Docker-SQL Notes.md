To replace the default long prompt with a >, we can type ` echo 'PS1="> "' > ~/.bashrc ` in the terminal. 


### What is Docker? ### 
- Good article:  https://www.geeksforgeeks.org/devops/introduction-to-docker/
Motivation for docker: before docker, deploying applications across environments was difficult, often encountering 'it works on my machine' problems due to dependencies, configs and OS-variations. 
How docker solves this problem: 
- Docker standardizes the runtime environment by bundling everything (app + dependencies) into a single, immutable container.
- Docker is an OS-level virtualization. Different to VMs that virutalize hardware and runs multiple guest OSs, Docker shares the host kernel and only isolates the specific software stack required to run the application.  
