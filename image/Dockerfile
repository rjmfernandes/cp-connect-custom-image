FROM confluentinc/cp-server-connect:7.6.2-3-ubi8
USER root
RUN pip3 uninstall -y setuptools
RUN pip3 uninstall -y pip
RUN rpm -e --nodeps platform-python-pip-9.0.3-24.el8.noarch
RUN rpm -e --nodeps python3-pip-wheel-9.0.3-24.el8.noarch
RUN rpm -e --nodeps python39-pip-20.2.4-9.module+el8.10.0+21329+8d76b841.noarch
RUN rpm -e --nodeps python39-pip-wheel-20.2.4-9.module+el8.10.0+21329+8d76b841.noarch
RUN rpm -e --nodeps platform-python-setuptools-39.2.0-7.el8.noarch
RUN rpm -e --nodeps python3-setuptools-wheel-39.2.0-7.el8.noarch
RUN rpm -e --nodeps python39-setuptools-wheel-50.3.2-5.module+el8.10.0+20345+671a55aa.noarch
