FROM ubuntu:22.04

RUN apt update
RUN apt install -y vim && apt install -y curl
RUN apt install -y git
RUN apt install -y net-tools
RUN apt install -y openssh-server
RUN apt install -y build-essential
RUN apt install -y gdb
RUN apt install -y g++
RUN apt install -y gcc
RUN apt install -y golang
RUN apt install -y make
RUN echo "root:wutianyu" | chpasswd
RUN ssh-keygen -A
RUN mkdir -p /var/run/sshd

EXPOSE 22 8080
