# Dockerfile
FROM ubuntu:20.04

# Install SSH server and other necessary packages
RUN apt-get update && apt-get install -y \
    openssh-server \
    sudo \
    && rm -rf /var/lib/apt/lists/*

# Create vagrant user with password 'vagrant'
RUN useradd -m -s /bin/bash vagrant && \
    echo "vagrant:vagrant" | chpasswd && \
    echo "vagrant ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/vagrant

# Set up SSH
RUN mkdir /var/run/sshd && \
    mkdir /home/vagrant/.ssh && \
    chmod 700 /home/vagrant/.ssh && \
    chown vagrant:vagrant /home/vagrant/.ssh

# Add Vagrant insecure public key
RUN echo "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key" > /home/vagrant/.ssh/authorized_keys

# Set correct permissions for authorized_keys
RUN chmod 600 /home/vagrant/.ssh/authorized_keys && \
    chown vagrant:vagrant /home/vagrant/.ssh/authorized_keys

# Start SSH service
CMD ["/usr/sbin/sshd", "-D"]

# Expose SSH port
EXPOSE 22

# Expose Nginx port
EXPOSE 80

# Expose Nginx port
EXPOSE 4444
