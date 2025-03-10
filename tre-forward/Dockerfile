FROM public.ecr.aws/lambda/python:3.13
ARG PIP_INDEX_URL

# Install dependancies 
RUN dnf upgrade
RUN dnf clean all --enablerepo=\*

# Copy function code
COPY tre_forward.py ${LAMBDA_TASK_ROOT}

# Set the CMD to your handler (could also be done as a parameter override outside of the Dockerfile)
CMD [ "tre_forward.lambda_handler" ]
