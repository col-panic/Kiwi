Running Kiwi TCMS as a Docker container
=========================================

In order to run Kiwi TCMS as a production instance you will need
`Docker <https://docs.docker.com/engine/installation/>`_ and
`docker-compose <https://docs.docker.com/compose/install/>`_. Refer to
their documentation about download and installation options.

Pull or Build Docker image
--------------------------

You can download the official Kiwi TCMS Docker image by running::

    docker pull kiwitcms/kiwi

Alternatively you can build an image yourself by running::

    make docker-image

this will create a Docker image with the latest Kiwi TCMS version.
By default the image tag will be ``kiwitcms/kiwi:latest``.

.. note::

    While Kiwi TCMS uses git tags when releasing new versions we do not
    provide versioned docker images via Docker Hub!


Start Docker compose
--------------------

Before starting Kiwi TCMS you need to clone the git repo::

    git clone https://github.com/kiwitcms/Kiwi.git


Then you can start Kiwi TCMS by executing::

    cd Kiwi/
    docker-compose up -d


Your Kiwi TCMS instance will be accessible at https://localhost.

The above command will create two containers:

1) A web container based on the latest Kiwi TCMS image
2) A DB container based on the official centos/mariadb image


``docker-compose`` will also create two volumes for persistent data storage:
``kiwi_db_data`` and ``kiwi_uploads``.

.. note::

    Kiwi TCMS container will bind to all network addresses on the system.
    To use it across the organization simply distribute the FQDN of the system
    running the Docker container to all associates.


Initial configuration of running container
------------------------------------------

You need to create the database schema by executing::

    docker exec -it kiwi_web /Kiwi/manage.py migrate

.. note::

    By default the first registered account will become superuser!

.. warning::

    This requires working email because the account must be activated via
    confirmation link sent to the email address defined during registration.

    If email is not configured or you prefer the command line use::

        docker exec -it kiwi_web /Kiwi/manage.py createsuperuser


Upgrading
---------

To upgrade running Kiwi TCMS containers execute the following commands::

    cd Kiwi/
    git pull # to refresh docker-compose.yml
    docker-compose down
    # make docker-image if you build from source or
    docker pull kiwitcms/kiwi  # to fetch latest version from Docker Hub
    docker pull centos/mariadb # to fetch the latest version for MariaDB
    docker-compose up -d
    docker exec -it kiwi_web /Kiwi/manage.py migrate

.. note::

    Uploads and database data should stay intact because they are split into
    separate volumes which makes upgrading very easy. However you may want to
    back these up before upgrading!


SSL configuration
-----------------

By default Kiwi TCMS is served via HTTPS. ``docker-compose.yml`` is configured with
a default self-signed certificate stored in ``etc/kiwitcms/ssl/``. If you want to
use different SSL certificate you need to update the ``localhost.key`` and
``localhost.crt`` files in that directory or bind-mount your own SSL directory to
``/Kiwi/ssl`` inside the docker container!

More information about generating your own self-signed certificates can be found at
https://wiki.centos.org/HowTos/Https.


Reverse proxy SSL
-----------------

To run Kiwi TCMS behind a reverse proxy, you may either directly pass HTTPS requests,
or allow Kiwi TMCS to serve via HTTP. With the later being the preferred method, when
the reverse proxy is terminating the SSL connection.

Passing HTTPS to the container
------------------------------

For all of these domains the browser will see a wildcard SSL certificate for
``*.kiwitcms.org``, while the individual docker containers are still configured
with the default self-signed certificate! Here's how the configuration looks like::

    http {
        # default ssl certificates for *.kiwitcms.org
        ssl_certificate     /etc/nginx/wildcard_kiwitcms_org.crt;
        ssl_certificate_key /etc/nginx/wildcard_kiwitcms_org.key;
    
        # default proxy settings
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    
        server {
            listen 8080;
            server_name demo.kiwitcms.org;
    
            location / {
                return 301 https://$host$request_uri;
            }
        }
    
        server {
            server_name demo.kiwitcms.org;
            listen 8443 ssl;
    
            location / {
                proxy_pass https://demo_kiwitcms_org_web:8443;
            }
        }
    }

Here is an equivalent configuration for `HAProxy <https://www.haproxy.org/>`_::

    frontend front_http
        bind *:8080
        reqadd X-Forwarded-Proto:\ http
        redirect scheme https code 301
    
    frontend front_https
        # default ssl certificates for *.kiwitcms.org
        bind *:8443 ssl crt /etc/haproxy/ssl/
        reqadd X-Forwarded-Proto:\ https
    
        acl kiwitcms hdr(host) -i demo.kiwitcms.org
        use_backend back_kiwitcms if kiwitcms
    
    backend back_kiwitcms
        http-request set-header X-Forwarded-Port %[dst_port]
        http-request add-header X-Forwarded-Proto https
    
        # some security tweaks
        rspadd Strict-Transport-Security:\ max-age=15768000
        rspadd X-XSS-Protection:\ 1;\ mode=block
    
        # do not verify the self-signed cert
        server kiwi_web demo_kiwitcms_org_web:8443 ssl verify none

Allow Kiwi TMCS HTTP access
---------------------------

By default the Kiwi TCMS container enforces HTTPS connections, by redirecting HTTP
requests to the HTTP port. This behavior may be deactivated, by starting the container
adding ```-e KIWI_DONT_ENFORCE_HTTPS=true```.

Customization
-------------

``tcms/settings/product.py`` contains the following lines at the end::

    try:
        from .local_settings import *  # noqa: F401,F403
    except ImportError:
        pass

This means you can edit ``docker-compose.yml`` to mount the host file
``local_settings.py`` inside the running Docker container::

        volumes:
            - uploads:/Kiwi/uploads
            - ./local_settings.py:/venv/lib64/python3.6/site-packages/tcms/settings/local_settings.py

essentially overriding any stock settings in this way!

.. warning::

    Some older versions of docker do not allow mounting of files between the
    host and the container, they only allow mounting directories and volumes.
    The stock docker versions on CentOS 7 and RHEL 7 do this. You may see an
    error similar to:

    ERROR: for kiwi_web Cannot start service web:
        OCI runtime create failed: container_linux.go:348:
            starting container process caused "process_linux.go:402:
                container init caused "rootfs_linux.go:58: mounting
                    "/root/kiwi/local_settings.py" to
                    rootfs "/var/lib/docker/overlay2 ....

    In this case you will either have to upgrade your docker version
    or ``COPY`` the desired files and rebuild the docker image!


Customized docker image
-----------------------

You can build your own customized version of Kiwi TCMS by adjusting
the contents of ``Dockerfile`` and then::

    make docker-image

.. note::

    Make sure to modify ``Makefile`` and ``docker-compose.yml`` to use your
    customized image name instead the default ``kiwitcms/kiwi:latest``!

.. warning::

    Modifying the default ``Dockerfile`` directly is not recommended because
    it is kept under version control and will start conflicting the next time
    you do ``git pull``. It is also not a very good idea to deploy an image built
    directly from the master branch.

    The proper way to create a downstream docker image is to provide a
    ``Dockerfile.myorg`` which inherits ``FROM kiwitcms/kiwi:latest``
    and adds your changes as separate layers! Ideally you will keep this into
    another git repository together with a ``Makefile`` and possibly your customized
    ``docker-compose.yml``.


Troubleshooting
----------------

The Kiwi TCMS container will print HTTPD logs on the docker console!

.. warning::

    You must start the containers in the foreground with ``docker-compose up``,
    e.g. without the ``-d`` option in order to see their logs!

In case you see a 500 Internal Server Error page and the error log does not
provide a traceback you should configure the ``DEBUG`` setting to ``True`` and
restart the docker container. If your changes are picked up correctly you
should see an error page with detailed information about the error instead of
the default 500 error page.

When reporting issues please copy the relevant traceback as plain text into
your reports!
