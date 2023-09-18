FROM ubuntu:latest
RUN apt update && apt install  openssh-server sudo ansible vim -y
# Copy the ssh public key in the authorized_keys file. The idkey.pub below is a public key file you get from ssh-keygen. They are under ~/.ssh directory by default.
RUN sudo su
RUN mkdir /ansible
RUN mkdir /root/.ssh
RUN chmod 700 /root/.ssh && chown root:root /root/.ssh


COPY id_rsa.pub /root/.ssh/authorized_keys
COPY id_rsa /root/.ssh/id_rsa
RUN chmod 600 /root/.ssh/id_rsa
RUN chmod 600 /root/.ssh/authorized_keys && chown root:root /root/.ssh/authorized_keys

WORKDIR /
EXPOSE 22
RUN service ssh start

CMD ["/usr/sbin/sshd","-D"]


# CMD ["/usr/lib/systemd/systemd"]
# CMD ["/usr/sbin/sshd","-D"]
