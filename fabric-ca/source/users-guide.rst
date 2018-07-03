Fabric CA 用户手册
======================

Hyperledger Fabric CA 就是Fabric的证书颁发机构(CA)。

它做以下这些事:

  * 身份信息注册, 或者连接到LDAP上创建用户信息；
  * 签发登记证书 (ECerts)；
  * 签发交易证书 (TCerts), 对在Fabric区块链上的交易实现不可连接性和匿名性；
  * 证书更新和吊销。

Fabric CA由一个客户端和一个服务端组成，我们一会讨论。

如果想对Fabric CA贡献你的开发能力, 请参见
`Fabric CA 仓库 <https://github.com/hyperledger/fabric-ca>`__ 。


.. _Back to Top:

内容大纲
-----------------

1. `概要说明`_

2. `开始`_

   1. `前提条件`_
   2. `安装`_
   3. `使用Fabric CA CLI`_

3. `文件格式`_

   1. `Fabric CA server的配置文件`_
   2. `Fabric CA client的配置文件`_

4. `几种配置方法及其优先级别`_

5. `Fabric CA 服务器`_

   1. `初始化服务器`_
   2. `启动服务器`_
   3. `配置数据库`_
   4. `配置LDAP`_
   5. `设置集群`_
   6. `设置多个CA`_
   7. `登记一个中级CA`_

6. `Fabric CA 客户端`_

   1. `登记bootstrap身份`_
   2. `注册一个新identity`_
   3. `登记一个peer身份`_
   4. `重新登记一个身份`_
   5. `注销一个证书或身份`_
   6. `启用TLS`_
   7. `Contact specific CA instance`_

概要说明
--------

下图展示了Fabric CA在整个架构里的位置。

.. image:: ./images/fabric-ca.png

有两种与fabric-CA server交互的方式:
fabric-CA client或fabric SDK
所有和CA服务器的通信都是Rest API
REST APIs请查看swagger 文档 `fabric-ca/swagger/swagger-fabric-ca.json` 

上图的右上角展示的是Fabric CA client 或 SDK 连接CA集群的情况。集群前面用HAProxy做负载，
客户端经过负载后连上集群中的一台服务器。

集群中所有的CA服务器都共享了同一个存放着身份信息和证书信息的数据库。
如果配置了LDAP，身份信息就不存在数据库上而是存在LDAP上了。

一台服务器上可能装有多个CA，每个CA可能是根CA或者是中级CA，每个中级CA都有一个上级CA，上级可能是根CA也有可能是另一个中级CA。

开始
---------------

前提条件
~~~~~~~~~~~~~~~

-  安装好 Go 1.7.x 
-  正确设置 ``GOPATH``环境变量 
-  安装好libtool 和 libtdhl-dev 包

在Ubuntu上安装libtool:

.. code:: bash

   sudo apt install libtool libltdl-dev

在MacOSX上安装libtool:

.. code:: bash

   brew install libtool

.. note:: libtldl-dev is not necessary on MacOSX if you instal
          libtool via Homebrew

关于libtool的更多信息 https://www.gnu.org/software/libtool.

关于libltdl-dev的更多信息 https://www.gnu.org/software/libtool/manual/html_node/Using-libltdl.html.

安装
~~~~~~~

安装 `fabric-ca-server` 和 `fabric-ca-client` 包到 $GOPATH/bin.

.. code:: bash

    go get -u github.com/hyperledger/fabric-ca/cmd/...

Note: 如果你已经 clone 了 fabric-ca 的repository， 要确保你是在
master branch 上，然后再执行上面这条 'go get' 命令， 否则你会看到如下错误:

::

    <gopath>/src/github.com/hyperledger/fabric-ca; git pull --ff-only
    There is no tracking information for the current branch.
    Please specify which branch you want to merge with.
    See git-pull(1) for details.

        git pull <remote> <branch>

    If you wish to set tracking information for this branch you can do so with:

        git branch --set-upstream-to=<remote>/<branch> tlsdoc

    package github.com/hyperledger/fabric-ca/cmd/fabric-ca-client: exit status 1

以原生方式启动CA服务器
~~~~~~~~~~~~~~~~~~~~~

用默认配置启动 `fabric-ca-server` 

.. code:: bash

    fabric-ca-server start -b admin:adminpw

这个 `-b` 提供了bootstrap管理员的 enrollment ID 和 secret ; 如果 "ldap.enabled" 设置没设置为true，则这个就必须要提供。

默认的配置文件名为 `fabric-ca-server-config.yaml`
它默认在本地目录下，但也可以自定义

用Docker方式启动CA服务器
~~~~~~~~~~~~~~~~~~~~~~~

Docker Hub
^^^^^^^^^^^^

到这里: https://hub.docker.com/r/hyperledger/fabric-ca/tags/

找到和你的fabric网络兼容的 fabric-ca 版本

到 `$GOPATH/src/github.com/hyperledger/fabric-ca/docker/server`
目录下打开 docker-compose.yml 文件。

修改 `image` ，改为你上面找的ca镜像的tag版本。下面这个示例是x86架构的beta镜像。

.. code:: yaml

    fabric-ca-server:
      image: hyperledger/fabric-ca:x86_64-1.0.0-beta
      container_name: fabric-ca-server
      ports:
        - "7054:7054"
      environment:
        - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      volumes:
        - "./fabric-ca-server:/etc/hyperledger/fabric-ca-server"
      command: sh -c 'fabric-ca-server start -b admin:adminpw'

在 docker-compose.yml 文件路径下，打开命令行
执行如下语句:

.. code:: bash

    # docker-compose up -d

这条命令会把镜像拉下来，然后启动fabric-ca服务器

编译你自己的 Docker 镜像
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

用如下命令编译并启动服务器。

.. code:: bash

    cd $GOPATH/src/github.com/hyperledger/fabric-ca
    make docker
    cd docker/server
    docker-compose up -d

hyperledger/fabric-ca docker 镜像包含了 fabric-ca-server 和 fabric-ca-client。

.. code:: bash

    # cd $GOPATH/src/github.com/hyperledger/fabric-ca
    # FABRIC_CA_DYNAMIC_LINK=true make docker
    # cd docker/server
    # docker-compose up -d

使用Fabric CA CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

下面展示了 Fabric CA server 命令的使用

.. code:: bash

    fabric-ca-server --help
    Hyperledger Fabric Certificate Authority Server

    Usage:
      fabric-ca-server [command]

    Available Commands:
      init        Initialize the Fabric CA server
      start       Start the Fabric CA server

    Flags:
      --address string                            Listening address of fabric-ca-server (default "0.0.0.0")
  -b, --boot string                               The user:pass for bootstrap admin which is required to build default config file
      --ca.certfile string                        PEM-encoded CA certificate file (default "ca-cert.pem")
      --ca.chainfile string                       PEM-encoded CA chain file (default "ca-chain.pem")
      --ca.keyfile string                         PEM-encoded CA key file (default "ca-key.pem")
  -n, --ca.name string                            Certificate Authority name
      --cacount int                               Number of non-default CA instances
      --cafiles stringSlice                       A list of comma-separated CA configuration files
  -c, --config string                             Configuration file (default "fabric-ca-server-config.yaml")
      --csr.cn string                             The common name field of the certificate signing request to a parent fabric-ca-server
      --csr.hosts stringSlice                     A list of comma-separated host names in a certificate signing request to a parent fabric-ca-server
      --db.datasource string                      Data source which is database specific (default "fabric-ca-server.db")
      --db.tls.certfiles stringSlice              A list of comma-separated PEM-encoded trusted certificate files (e.g. root1.pem,root2.pem)
      --db.tls.client.certfile string             PEM-encoded certificate file when mutual authenticate is enabled
      --db.tls.client.keyfile string              PEM-encoded key file when mutual authentication is enabled
      --db.type string                            Type of database; one of: sqlite3, postgres, mysql (default "sqlite3")
  -d, --debug                                     Enable debug level logging
      --intermediate.enrollment.label string      Label to use in HSM operations
      --intermediate.enrollment.profile string    Name of the signing profile to use in issuing the certificate
      --intermediate.parentserver.caname string   Name of the CA to connect to on fabric-ca-serve
  -u, --intermediate.parentserver.url string      URL of the parent fabric-ca-server (e.g. http://<username>:<password>@<address>:<port)
      --intermediate.tls.certfiles stringSlice    A list of comma-separated PEM-encoded trusted certificate files (e.g. root1.pem,root2.pem)
      --intermediate.tls.client.certfile string   PEM-encoded certificate file when mutual authenticate is enabled
      --intermediate.tls.client.keyfile string    PEM-encoded key file when mutual authentication is enabled
      --ldap.enabled                              Enable the LDAP client for authentication and attributes
      --ldap.groupfilter string                   The LDAP group filter for a single affiliation group (default "(memberUid=%s)")
      --ldap.tls.certfiles stringSlice            A list of comma-separated PEM-encoded trusted certificate files (e.g. root1.pem,root2.pem)
      --ldap.tls.client.certfile string           PEM-encoded certificate file when mutual authenticate is enabled
      --ldap.tls.client.keyfile string            PEM-encoded key file when mutual authentication is enabled
      --ldap.url string                           LDAP client URL of form ldap://adminDN:adminPassword@host[:port]/base
      --ldap.userfilter string                    The LDAP user filter to use when searching for users (default "(uid=%s)")
  -p, --port int                                  Listening port of fabric-ca-server (default 7054)
      --registry.maxenrollments int               Maximum number of enrollments; valid if LDAP not enabled
      --tls.certfile string                       PEM-encoded TLS certificate file for server's listening port (default "ca-cert.pem")
      --tls.clientauth.certfiles stringSlice      A list of comma-separated PEM-encoded trusted certificate files (e.g. root1.pem,root2.pem)
      --tls.clientauth.type string                Policy the server will follow for TLS Client Authentication. (default "noclientcert")
      --tls.enabled                               Enable TLS on the listening port
      --tls.keyfile string                        PEM-encoded TLS key for server's listening port (default "ca-key.pem")

    Use "fabric-ca-server [command] --help" for more information about a command.

以下展示了 Fabric CA client 命令的使用方法:

.. code:: bash

    fabric-ca-client --help
    Hyperledger Fabric Certificate Authority Client

    Usage:
      fabric-ca-client [command]

    Available Commands:
      enroll      Enroll an identity
      getcacert   Get CA certificate chain
      reenroll    Reenroll an identity
      register    Register an identity
      revoke      Revoke an identity

    Flags:
      --caname string                Name of CA
  -c, --config string                Configuration file (default "/Users/saadkarim/.fabric-ca-client/fabric-ca-client-config.yaml")
      --csr.hosts stringSlice        A list of comma-separated host names in a certificate signing request
      --csr.serialnumber string      The serial number in a certificate signing request, which becomes part of the DN (Distinquished Name)
  -d, --debug                        Enable debug level logging
      --enrollment.label string      Label to use in HSM operations
      --enrollment.profile string    Name of the signing profile to use in issuing the certificate
      --id.affiliation string        The identity's affiliation
      --id.attrs stringSlice         A list of comma-separated attributes of the form <name>=<value> (e.g. foo=foo1,bar=bar1)
      --id.maxenrollments int        The maximum number of times the secret can be reused to enroll.
      --id.name string               Unique name of the identity
      --id.secret string             The enrollment secret for the identity being registered
      --id.type string               Type of identity being registered (e.g. 'peer, app, user')
  -M, --mspdir string                Membership Service Provider directory (default "msp")
  -m, --myhost string                Hostname to include in the certificate signing request during enrollment (default "saads-mbp.raleigh.ibm.com")
  -a, --revoke.aki string            AKI (Authority Key Identifier) of the certificate to be revoked
  -e, --revoke.name string           Identity whose certificates should be revoked
  -r, --revoke.reason string         Reason for revocation
  -s, --revoke.serial string         Serial number of the certificate to be revoked
      --tls.certfiles stringSlice    A list of comma-separated PEM-encoded trusted certificate files (e.g. root1.pem,root2.pem)
      --tls.client.certfile string   PEM-encoded certificate file when mutual authenticate is enabled
      --tls.client.keyfile string    PEM-encoded key file when mutual authentication is enabled
  -u, --url string                   URL of fabric-ca-server (default "http://localhost:7054")

    Use "fabric-ca-client [command] --help" for more information about a command.

.. note:: Note that command line options that are string slices (lists) can be
          specified either by specifying the option with comma-separated list
          elements or by specifying the option multiple times, each with a
          string value that make up the list. For example, to specify
          ``host1`` and ``host2`` for the ``csr.hosts`` option, you can either
          pass ``--csr.hosts 'host1,host2'`` or
          ``--csr.hosts host1 --csr.hosts host2``. When using the former format,
          please make sure there are no space before or after any commas.

`Back to Top`_

文件格式
------------

Fabric CA server的配置文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

默认的配置文件 (如下所示) 是生成在服务器的 home 目录下的 (请见 `Fabric CA Server <#server>`__ 章节).

.. code:: yaml

    # Server's listening port (default: 7054)
    port: 7054

    # Enables debug logging (default: false)
    debug: false

    #############################################################################
    #  TLS section for the server's listening port
    #
    #  The following types are supported for client authentication: NoClientCert,
    #  RequestClientCert, RequireAnyClientCert, VerifyClientCertIfGiven,
    #  and RequireAndVerifyClientCert.
    #
    #  Certfiles is a list of root certificate authorities that the server uses
    #  when verifying client certificates.
    #############################################################################
    tls:
      # Enable TLS (default: false)
      enabled: false
      # TLS for the server's listening port
      certfile: ca-cert.pem
      keyfile: ca-key.pem
      clientauth:
        type: noclientcert
        certfiles:

    #############################################################################
    #  The CA section contains information related to the Certificate Authority
    #  including the name of the CA, which should be unique for all members
    #  of a blockchain network.  It also includes the key and certificate files
    #  used when issuing enrollment certificates (ECerts) and transaction
    #  certificates (TCerts).
    #  The chainfile (if it exists) contains the certificate chain which
    #  should be trusted for this CA, where the 1st in the chain is always the
    #  root CA certificate.
    #############################################################################
    ca:
      # Name of this CA
      name:
      # Key file (default: ca-key.pem)
      keyfile: ca-key.pem
      # Certificate file (default: ca-cert.pem)
      certfile: ca-cert.pem
      # Chain file (default: chain-cert.pem)
      chainfile: ca-chain.pem

    #############################################################################
    #  The registry section controls how the fabric-ca-server does two things:
    #  1) authenticates enrollment requests which contain a username and password
    #     (also known as an enrollment ID and secret).
    #  2) once authenticated, retrieves the identity's attribute names and
    #     values which the fabric-ca-server optionally puts into TCerts
    #     which it issues for transacting on the Hyperledger Fabric blockchain.
    #     These attributes are useful for making access control decisions in
    #     chaincode.
    #  There are two main configuration options:
    #  1) The fabric-ca-server is the registry
    #  2) An LDAP server is the registry, in which case the fabric-ca-server
    #     calls the LDAP server to perform these tasks.
    #############################################################################
    registry:
      # Maximum number of times a password/secret can be reused for enrollment
      # (default: -1, which means there is no limit)
      maxenrollments: -1

      # Contains identity information which is used when LDAP is disabled
      identities:
         - name: <<<ADMIN>>>
           pass: <<<ADMINPW>>>
           type: client
           affiliation: ""
           maxenrollments: -1
           attrs:
              hf.Registrar.Roles: "client,user,peer,validator,auditor"
              hf.Registrar.DelegateRoles: "client,user,validator,auditor"
              hf.Revoker: true
              hf.IntermediateCA: true

    #############################################################################
    #  Database section
    #  Supported types are: "sqlite3", "postgres", and "mysql".
    #  The datasource value depends on the type.
    #  If the type is "sqlite3", the datasource value is a file name to use
    #  as the database store.  Since "sqlite3" is an embedded database, it
    #  may not be used if you want to run the fabric-ca-server in a cluster.
    #  To run the fabric-ca-server in a cluster, you must choose "postgres"
    #  or "mysql".
    #############################################################################
    db:
      type: sqlite3
      datasource: fabric-ca-server.db
      tls:
          enabled: false
          certfiles:
            - db-server-cert.pem
          client:
            certfile: db-client-cert.pem
            keyfile: db-client-key.pem

    #############################################################################
    #  LDAP section
    #  If LDAP is enabled, the fabric-ca-server calls LDAP to:
    #  1) authenticate enrollment ID and secret (i.e. username and password)
    #     for enrollment requests;
    #  2) To retrieve identity attributes
    #############################################################################
    ldap:
       # Enables or disables the LDAP client (default: false)
       enabled: false
       # The URL of the LDAP server
       url: ldap://<adminDN>:<adminPassword>@<host>:<port>/<base>
       tls:
          certfiles:
            - ldap-server-cert.pem
          client:
             certfile: ldap-client-cert.pem
             keyfile: ldap-client-key.pem

    #############################################################################
    #  Affiliation section
    #############################################################################
    affiliations:
       org1:
          - department1
          - department2
       org2:
          - department1

    #############################################################################
    #  Signing section
    #
    #  The "default" subsection is used to sign enrollment certificates;
    #  the default expiration ("expiry" field) is "8760h", which is 1 year in hours.
    #
    #  The "ca" profile subsection is used to sign intermediate CA certificates;
    #  the default expiration ("expiry" field) is "43800h" which is 5 years in hours.
    #  Note that "isca" is true, meaning that it issues a CA certificate.
    #  A maxpathlen of 0 means that the intermediate CA cannot issue other
    #  intermediate CA certificates, though it can still issue end entity certificates.
    #  (See RFC 5280, section 4.2.1.9)
    #############################################################################
    signing:
        default:
          usage:
            - cert sign
          expiry: 8760h
        profiles:
          ca:
             usage:
               - cert sign
             expiry: 43800h
             caconstraint:
               isca: true
               maxpathlen: 0

    ###########################################################################
    #  Certificate Signing Request (CSR) section.
    #  This controls the creation of the root CA certificate.
    #  The expiration for the root CA certificate is configured with the
    #  "ca.expiry" field below, whose default value is "131400h" which is
    #  15 years in hours.
    #  The pathlength field is used to limit CA certificate hierarchy as described
    #  in section 4.2.1.9 of RFC 5280.
    #  Examples:
    #  1) No pathlength value means no limit is requested.
    #  2) pathlength == 1 means a limit of 1 is requested which is the default for
    #     a root CA.  This means the root CA can issue intermediate CA certificates,
    #     but these intermediate CAs may not in turn issue other CA certificates
    #     though they can still issue end entity certificates.
    #  3) pathlength == 0 means a limit of 0 is requested;
    #     this is the default for an intermediate CA, which means it can not issue
    #     CA certificates though it can still issue end entity certificates.
    ###########################################################################
    csr:
       cn: <<<COMMONNAME>>>
       names:
          - C: US
            ST: "North Carolina"
            L:
            O: Hyperledger
            OU: Fabric
       hosts:
         - <<<MYHOST>>>
         - localhost
       ca:
          expiry: 131400h
          pathlength: <<<PATHLENGTH>>>

    #############################################################################
    # BCCSP (BlockChain Crypto Service Provider) section is used to select which
    # crypto library implementation to use
    #############################################################################
    bccsp:
        default: SW
        sw:
            hash: SHA2
            security: 256
            filekeystore:
                # The directory used for the software file-based keystore
                keystore: msp/keystore

    #############################################################################
    # Multi CA section
    #
    # Each Fabric CA server contains one CA by default.  This section is used
    # to configure multiple CAs in a single server.
    #
    # 1) --cacount <number-of-CAs>
    # Automatically generate <number-of-CAs> non-default CAs.  The names of these
    # additional CAs are "ca1", "ca2", ... "caN", where "N" is <number-of-CAs>
    # This is particularly useful in a development environment to quickly set up
    # multiple CAs.
    #
    # 2) --cafiles <CA-config-files>
    # For each CA config file in the list, generate a separate signing CA.  Each CA
    # config file in this list MAY contain all of the same elements as are found in
    # the server config file except port, debug, and tls sections.
    #
    # Examples:
    # fabric-ca-server start -b admin:adminpw --cacount 2
    #
    # fabric-ca-server start -b admin:adminpw --cafiles ca/ca1/fabric-ca-server-config.yaml
    # --cafiles ca/ca2/fabric-ca-server-config.yaml
    #
    #############################################################################

    cacount:

    cafiles:

    #############################################################################
    # Intermediate CA section
    #
    # The relationship between servers and CAs is as follows:
    #   1) A single server process may contain or function as one or more CAs.
    #      This is configured by the "Multi CA section" above.
    #   2) Each CA is either a root CA or an intermediate CA.
    #   3) Each intermediate CA has a parent CA which is either a root CA or another intermediate CA.
    #
    # This section pertains to configuration of #2 and #3.
    # If the "intermediate.parentserver.url" property is set,
    # then this is an intermediate CA with the specified parent
    # CA.
    #
    # parentserver section
    #    url - The URL of the parent server
    #    caname - Name of the CA to enroll within the server
    #
    # enrollment section used to enroll intermediate CA with parent CA
    #    profile - Name of the signing profile to use in issuing the certificate
    #    label - Label to use in HSM operations
    #
    # tls section for secure socket connection
    #   certfiles - PEM-encoded list of trusted root certificate files
    #   client:
    #     certfile - PEM-encoded certificate file for when client authentication
    #     is enabled on server
    #     keyfile - PEM-encoded key file for when client authentication
    #     is enabled on server
    #############################################################################
    intermediate:
      parentserver:
        url:
        caname:

      enrollment:
        hosts:
        profile:
        label:

      tls:
        certfiles:
        client:
          certfile:
          keyfile:

Fabric CA client的配置文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下默认配置文件生成在 client的home 目录下 (请见 `Fabric CA Client <#client>`__ 章节).

.. code:: yaml

    #############################################################################
    # Client Configuration
    #############################################################################

    # URL of the Fabric CA server (default: http://localhost:7054)
    URL: http://localhost:7054

    # Membership Service Provider (MSP) directory
    # When the client is used to enroll a peer or an orderer, this field must be
    # set to the MSP directory of the peer/orderer
    MSPDir:

    #############################################################################
    #    TLS section for secure socket connection
    #############################################################################
    tls:
      # Enable TLS (default: false)
      enabled: false
      certfiles:
      client:
        certfile:
        keyfile:

    #############################################################################
    #  Certificate Signing Request section for generating the CSR for
    #  an enrollment certificate (ECert)
    #############################################################################
    csr:
      cn: <<<ENROLLMENT_ID>>>
      names:
        - C: US
          ST: North Carolina
          L:
          O: Hyperledger
          OU: Fabric
      hosts:
       - <<<MYHOST>>>
      ca:
        pathlen:
        pathlenzero:
        expiry:

    #############################################################################
    #  Registration section used to register a new identity with Fabric CA server
    #############################################################################
    id:
      name:
      type:
      affiliation:
      attributes:
        - name:
          value:

    #############################################################################
    #  Enrollment section used to enroll an identity with Fabric CA server
    #############################################################################
    enrollment:
      profile:
      label:

    # Name of the CA to connect to within the fabric-ca server
    caname:

大吉注：
client配置文件主要包括如下内容：
- MSPDir：设置要enroll的peer/orderer的MSP文件夹
- csr：为一个ECert生成一个CSR
- registration section（id）：register一个新的identity
- enrollment section（enrollment）：enroll一个identity
- caname：连接的ca的名字

`Back to Top`_

几种配置方法及其优先级别
---------------------------------

Fabric CA 有三种对配置进行设置的方法（优先级由大到小）： :

  1. CLI 参数
  2. 环境变量
  3. 配置文件

以下内容中，我们将演示如何改配置文件，但是配置文件的设置会被环境变量及CLI参数覆盖。

如下是client的配置文件:

.. code:: yaml

    tls:
      # Enable TLS (default: false)
      enabled: false

      # TLS for the client's listenting port (default: false)
      certfiles:
      client:
        certfile: cert.pem
        keyfile:

下面的环境变量将会覆盖上面的配置:

.. code:: bash

  export FABRIC_CA_CLIENT_TLS_CLIENT_CERTFILE=cert2.pem

下面这个CLI参数能覆盖配置文件和环境变量：

.. code:: bash

  fabric-ca-client enroll --tls.client.certfile cert3.pem

fabric-ca-server服务器也一样， 只不过环境变量名不是以
``FABIRC_CA_CLIENT`` 开头,而是
``FABRIC_CA_SERVER`` 。

.. _server:


一句话解释文件路径
--------------------

CA服务器端或客户端配置文件中，所有文件属性都可设置为绝对或相对路径。

相对路径是相对于配置文件所在目录。比如, 如果配置文件在
``~/config`` 目录下，下面这个配置信息里的 ``root.pem`` 就应该在 ``~/config``
目录下， ``cert.pem`` 文件则在 ``~/config/certs`` 目录下，
``key.pem`` 文件在 ``/abs/path`` 目录下。

.. code:: yaml

    tls:
      enabled: true
      certfiles:
        - root.pem
      client:
        certfile: certs/cert.pem
        keyfile: /abs/path/key.pem



Fabric CA 服务器
----------------

这一节讲的是CA服务器。

在启动服务器前要先初始化它。这个过程会产生一份默认的配置文件，然后你可以review，修改。

Fabric CA服务器的home目录是这样决定的:
  - 如果设置了 ``FABRIC_CA_SERVER_HOME`` 环境变量, 则就取它的值
  - 否则就取 ``FABRIC_CA_HOME`` 的值
  - 否则就取 ``CA_CFG_PATH`` 的值
  - 否则就用当前的工作目录
  
这个章节的剩余部分, 我们假设你已经设置了环境变量 ``FABRIC_CA_HOME`` 为
``$HOME/fabric-ca/server``。

下面的指令假设你已经将配置文件放在了服务器的home目录下.

.. _initialize:

初始化服务器
~~~~~~~~~~~~~~~~~~~~~~~

用以下语句初始化CA服务器:

.. code:: bash

    fabric-ca-server init -b admin:adminpw

当LDAP被禁用时，就必须要有这个 ``-b`` (代表“启动身份”bootstrap identity) 选项。 启动服务器必须要有启动身份; 这个身份就是管理员身份。

配置文件里可以配置证书签名请求 (CSR)域
以下就是一个CSR域的示例。

.. _csr-fields:

.. code:: yaml

   cn: fabric-ca-server
   names:
      - C: US
        ST: "North Carolina"
        L:
        O: Hyperledger
        OU: Fabric
   hosts:
     - host1.example.com
     - localhost
   ca:
      expiry: 131400h
      pathlength: 1


以上所有字段都对应了X.509证书的字段，即调用 ``fabric-ca-server init`` 生成的证书字段。
这个CSR的域设置效果等同于配置中的 ``ca.certfile`` 和 ``ca.keyfile`` 两个配置域的组合。
（大吉注：配置了CSR域就是用这些信息自己给自己签名，ca.certfile和ca.keyfile是用这两个文件自签名） 
字段解释如下:

  -  **cn** 证书名Common Name
  -  **O** 组织名organization name
  -  **OU** 组织单元organizational unit
  -  **L** 位置location or city
  -  **ST** 州state
  -  **C** 国家country

如果要配置CSR，就要把 ``ca.certfile`` 和 ``ca-keyfile`` 对应的文件删了。（官方默认是ca-cert.pem和ca-key.pem）
然后重新运行一下 ``fabric-ca-server init -b admin:adminpw``

 ``fabric-ca-server init`` 命令会生成一个自签名证书除非你设置了 ``-u <parent-fabric-ca-server-URL>`` 选项。
如果指定了 ``-u`` 则CA证书将由上级CA签发。
为了得到上级Fabric CA 服务器认证，URL格式必须是以 ``<scheme>://<enrollmentID>:<secret>@<host>:<port>``， 其中
<enrollmentID> 和 <secret> 指代了一个 'hf.IntermediateCA'为true（大吉注：即可登记中间服务器）的身份。
命令 ``fabric-ca-server init`` 会生成一个默认文件 **fabric-ca-server-config.yaml** 到home目录下。

如果你要指定 CA 签名证书 和 key 文件，
你就得把文件放到 ``ca.certfile`` 和 ``ca.keyfile`` 的指定路径下。
文件必须是PEM格式且不可加密。
CA签名证书必须以 ``-----BEGIN CERTIFICATE-----`` 开头。
key 文件必须以 ``-----BEGIN PRIVATE KEY-----`` 开头，而不是
``-----BEGIN ENCRYPTED PRIVATE KEY-----``。

算法和key长度

CSR 域能自定义支持椭圆曲线算法(ECDSA)的 X.509 证书和key。 
以下是一个示例实现了椭圆曲线数字签名算法(ECDSA) ，曲线是 ``prime256v1`` 签名算法是
``ecdsa-with-SHA256``:

.. code:: yaml

    key:
       algo: ecdsa
       size: 256

按自己的安全级别指定算法和key长度。

椭圆曲线 (ECDSA) 提供以下 key 长度选项:

+--------+--------------+-----------------------+
| 长度   | 曲线标识符   |     签名算法          |
+========+==============+=======================+
| 256    | prime256v1   | ecdsa-with-SHA256     |
+--------+--------------+-----------------------+
| 384    | secp384r1    | ecdsa-with-SHA384     |
+--------+--------------+-----------------------+
| 521    | secp521r1    | ecdsa-with-SHA512     |
+--------+--------------+-----------------------+

启动服务器
~~~~~~~~~~~~~~~~~~~

用以下命令启动CA服务器:

.. code:: bash

    fabric-ca-server start -b <admin>:<adminpw>

第一次启动时，如果服务器未初始化，则会先进行初始化。在初始化期间，如果发现
ca-cert.pem 和 ca-key.pem 不存在，则会先生成，如果配置文件不存在也会生成默认的配置文件。
查看 `初始化服务器 <#initialize>`__ 章节.

除非你用的是LDAP，否则你必须要先有一个预先注册好的bootstrap身份信息用来注册和登记其他身份信息。
用 ``-b`` 选项来指定bootstrap身份。

如果要让服务器监听 ``https`` 而不是 ``http``，则需要设置 ``tls.enabled`` 为 ``true``。

要限制同一个 secret (或 password) 的登记使用次数，需要给 ``registry.maxenrollments`` 配置项设置一个值。
如果设置为1, 则每个 enrollment ID只能被登记一次，如果设置为 -1, 则secret的登记使用次数不做限制。
默认值是-1。 如果设置为0, 则所有的身份或者是注册进来的身份都不能被登记了。

启动后，CA服务器监听端口是 7054。

你可以跳到 `客户端 <#fabric-ca-client>`__ 章节如果你不想把CA服务器配置为集群或者使用LDAP.

配置数据库
~~~~~~~~~~~~~~~~~~~~~~~~


这一章节描述如何配置CA服务器连接PostgreSQL或者MySQL数据库。
默认的数据库是SQLite，默认的数据库文件是home目录下的 ``fabric-ca-server.db``。 

如果你不关心如何配置CA服务器集群，你也可以跳过这一章。

PostgreSQL
^^^^^^^^^^

以下是PostgreSQL的配置示例，具体请参考:
https://www.postgresql.org/docs/current/static/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS

.. code:: yaml

    db:
      type: postgres
      datasource: host=localhost port=5432 user=Username password=Password dbname=fabric_ca sslmode=verify-full

*sslmode* 指定了 SSL模式。 以下是各SSL模式说明:

|

+----------------+----------------+
| 模式名         | 描述           |
+================+================+
| disable        | 不启用SSL      |
+----------------+----------------+
| require        | 永远启用SSL，  |
|                | 不校验数据库   | 
|                | 服务端证书     |
+----------------+----------------+
| verify-ca      | 永远启用SSL，  |
|                | 校验数据库     |
|                | 服务端证书，   |
|                | 看其是否是由   |
|                | 可信CA签发的。 |
|                |                |
+----------------+----------------+
| verify-full    | 和             |
|                | verify-ca 类似 |
|                | ，同样校验     |
|                | 数据库服务端   |
|                | 证书，并且     |
|                | 服务端的       |
|                | hostname必须   |
|                | 与证书里的     |
|                | hostname一致。 |
+----------------+----------------+

|

若你要使用TLS，需要在配置文件中指定 ``db.tls`` 域， 若启用客户端校验, 
需在 ``db.tls.client`` 域指定客户端证书和客户端key文件（私钥）。
以下是 ``db.tls`` 域的配置示例:

.. code:: yaml

    db:
      ...
      tls:
          enabled: true
          certfiles:
            - db-server-cert.pem
          client:
                certfile: db-client-cert.pem
                keyfile: db-client-key.pem

| **certfiles** - PEM格式的可信根证书列表.
| **certfile** 和 **keyfile** - PEM格式的证书和key文件，用于CA服务器与PostgreSQL的通信。

PostgreSQL SSL 配置
"""""""""""""""""""""""""""""

**PostgreSQL 服务器SSL的基础配置步骤:**

1. 在postgresql.conf, 去掉SSL的注释并设置为 "on" (SSL=on)

2. 将证书和key文件放在PostgreSQL的data目录。

生成自签名证书的步骤:
https://www.postgresql.org/docs/9.5/static/ssl-tcp.html

注意: 自签名证书建议只用在测试环境，别用在生产环境。

**PostgreSQL 服务器 - 客户端证书校验配置**

1. 将可信CA写入data目录下的 root.crt 中

2. 打开postgresql.conf, 设置 "ssl\_ca\_file" 设置客户端证书的根证

3. 打开pg\_hba.conf，在1或多条hostssl中设置clientcert 参数为 1

更多PostgreSQL配置请见:
https://www.postgresql.org/docs/9.4/static/libpq-ssl.html

MySQL
^^^^^^^

以下示例可以用于CA服务器配置以启用MySQL为数据库服务，更多配置请参考:
https://dev.mysql.com/doc/refman/5.7/en/identifiers.html

MySQL 5.7.X中，若想让服务器接受’0000-00-00’为有效的日期，需要在my.cnf中找到配置选项*sql_mode*，然后删除*NO_ZERO_DATE*，并重启服务器。

具体的设置选择可参考 
https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html


.. code:: yaml

    db:
      type: mysql
      datasource: root:rootpw@tcp(localhost:3306)/fabric_ca?parseTime=true&tls=custom

若要通过TLS连接MySQL服务器，``db.tls.client``的设置参考上文 **PostgreSQL**的设置。

MySQL SSL 配置
""""""""""""""""""""""""

**MySQL 服务器SSL配置的基础步骤:**

1. 打开或创建 my.cnf 文件， 添加或反注释 [mysqld] 域。 指定服务器密钥，证书，及ca证书.

   生成服务端和客户端证书的步骤:
   http://dev.mysql.com/doc/refman/5.7/en/creating-ssl-files-using-openssl.html

   [mysqld] ssl-ca=ca-cert.pem ssl-cert=server-cert.pem ssl-key=server-key.pem

   调用以下查询SQL确认SSL 已经被开启。

   mysql> SHOW GLOBAL VARIABLES LIKE 'have\_%ssl';

   正常结果应该如下:

   +----------------+----------------+
   | Variable_name  | Value          |
   +================+================+
   | have_openssl   | YES            |
   +----------------+----------------+
   | have_ssl       | YES            |
   +----------------+----------------+

2. 完成服务端SSL配置后, 下一步是创建一个有权限使用SSL访问数据库的用户，
   先登录MySQL 服务器，然后输入如下语句:

   mysql> GRANT ALL PRIVILEGES ON *.* TO 'ssluser'@'%' IDENTIFIED BY
   'password' REQUIRE SSL; mysql> FLUSH PRIVILEGES;

   如果你需要指定限制客户端允许的IP，则需将 '%' 改为指定的客户端IP.

**MySQL Server - 客户端证书校验配置**

安全连接的选项和用于服务器端的选项是相似的
- ssl-ca 识别CA证书，如果用到，必须和服务器端用相同的证书。
- ssl-cert 识别MySQL服务器的证书。
- ssl-key 识别MySQL服务器的私钥。

假设你想要用一个账户来连接，这个账户没有特殊的加密要求或是被一个包括REQUIRE SSL的GRANT statement创建的，
启动MySQL服务至少需要-ssl-cert和-ssl-key选项。然后在服务设置文件中设置 ``db.tls.certfiles``属性并启动CA服务。

为了要求一个客户端证书也是被指定的，创建一个用REQUIRE X509选项的账户。
然后客户端也必须明确客户端密钥和证书文件；否则，MySQL server将会拒绝连接。
为了给CA server指定客户端密钥和证书文件，设置  ``db.tls.client.certfile``和 ``db.tls.client.keyfile``

配置LDAP
~~~~~~~~~~~~~~~~

CA server可以从LDAP server中读取。
特别地，CA server可以和一个LDAP server连接做如下事情：
- 认证一个identity去enrollment的优先级
- 检索一个identity用于认证的属性

修改CA server配置文件的LDAP部分来配置CA server连接LDAP服务器

.. code:: yaml

    ldap:
       # Enables or disables the LDAP client (default: false)
       enabled: false
       # The URL of the LDAP server
       url: <scheme>://<adminDN>:<adminPassword>@<host>:<port>/<base>
       userfilter: filter

其中:

  * ``scheme`` *ldap* 或 *ldaps*中的一个;
  * ``adminDN`` admin用户的名字；
  * ``pass`` admin用户的密码。
  * ``host`` LDAP服务器的hostname或IP地址。
  * ``port`` 可选的端口号，ldap默认的端口号是389，ldaps默认的端口号是636。
  * ``base`` LDAP树的根，用来搜索。
  * ``filter`` 登录名的过滤器，例如：(uid=%s)用来搜索用用户名登陆的用户，(email=%s)用来搜索用邮箱登陆的用户。	
	
	
以下是OpenLDAP server默认配置的样例
OpenLDAP server的docker 镜像文件在
``https://github.com/osixia/docker-openldap``.

.. code:: yaml

    ldap:
       enabled: true
       url: ldap://cn=admin,dc=example,dc=org:admin@localhost:10389/dc=example,dc=org
       userfilter: (uid=%s)

可查看``FABRIC_CA/scripts/run-ldap-tests`` 这个测试脚本，它的测试步骤是：
启动OpenLDAP的docker镜像，然后对它进行配置，
接着运行``FABRIC_CA/cli/server/ldap/ldap_test.go`` 以测试LDAP，
最后停止OpenLDAP服务器。

当LDAP被配置好之后，enrollment过程如下：

-  CA client或client SDK发送一个enrollment请求，这个请求带一个basic authorization header。
-  CA server接收了这个enrollment请求，对authorization header中的identity name和password进行解码，
   用配置文件中的“userfilter”来从identity name中查找DN（Distinquished Name），
   然后用identity的密码请求一个LDAP bind，如果LDAP bind成功了，enrollment过程就被授权了，可以执行了。

当LDAP被配置好之后，提取属性（attribute retrieval）的过程如下：

-  client SDK给CA server发送一个对一批tcerts的请求 **用一个或多个attributes**。
-  CA server接收这个tcert请求，并完成如下步骤：
   
   -  从authorization header的token（验证过token之后）中提取enrollment ID。
   -  向LDAP服务器发起一个LDAP搜索，查找tcert请求中的所有属性名。
   -  将属性值放在tcert中。   
   
设置集群
~~~~~~~~~~~~~~~~~~~~

配置Haproxy去平衡CA server集群中各个server的负载。确保更改hostname和port来对应CA server的设置。

haproxy.conf

.. code::

    global
          maxconn 4096
          daemon

    defaults
          mode http
          maxconn 2000
          timeout connect 5000
          timeout client 50000
          timeout server 50000

    listen http-in
          bind *:7054
          balance roundrobin
          server server1 hostname1:port
          server server2 hostname2:port
          server server3 hostname3:port


注意: 如果要用TLS，需要用 ``mode tcp``.

设置多个CA
~~~~~~~~~~~~~~~~~~~~~~~

fabric-ca server默认是一个单独的CA。可以通过 `cafiles`和 `cacount`配置选项来增加其他的CA，
每一个CA都有他自己的home directory。

cacount:
^^^^^^^^

 `cacount`可以直接设置additional CAs，他们的home directory和server directory相关，如下：

.. code:: yaml

    --<Server Home>
      |--ca
        |--ca1
        |--ca2

每个额外的CA将会在他的home directory里生成一个默认的配置文件，其中包括唯一的CA name。
如下命令来启动2个CA：

.. code:: bash

    fabric-ca-server start -b admin:adminpw --cacount 2

cafiles:
^^^^^^^^

如果cafiles未使用绝对路径，则CA的home目录将相对于服务器目录存在。

为了使用这个选项，CA配置文件必须已经生成好了，给每个将启动的CA配置好了。
每个配置文件必须有唯一的CA名称，和Common Name（CN），否则服务器会启动失败。
CA配置文件里，每个配置项内容将覆盖默认配置内容，空缺的配置项内容将默认为默认配置的内容。

配置的优先级如下:

  1. CA 配置文件
  2. 默认的CA CLI 标记
  3. 默认的 CA 环境变量
  4. 默认的 CA 配置文件

一个CA的配置文件至少包括下述内容：

.. code:: yaml

    ca:
    # Name of this CA
    name: <CANAME>

    csr:
      cn: <COMMONNAME>

可以设置文档结构如下：

.. code:: yaml

    --<Server Home>
      |--ca
        |--ca1
          |-- fabric-ca-config.yaml
        |--ca2
          |-- fabric-ca-config.yaml

如下命令可以启动两个定制化CA实例：

.. code:: bash

    fabric-ca-server start -b admin:adminpw --cafiles ca/ca1/fabric-ca-config.yaml
    --cafiles ca/ca2/fabric-ca-config.yaml

登记一个中级CA
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了给中级CA创建一个CA签名证书，中级CA必须向父CA登记，
与fabric-ca-client向CA登记申请证书操作一样。
登记需通过用 -u选项指定一个父CA的URL(包含enrollmentID和secret)，下面有示例。
enrollmentID对应的身份属性"hf.IntermediateCA"必须为"true"。
申请到的证书的CN隐性地就是enrollmentID。
如果你显性地指定了CN，将会报错。


.. code:: bash

    fabric-ca-server start -b admin:adminpw -u http://<enrollmentID>:<secret>@<parentserver>:<parentport>

其他中级CA 标记请查看 `Fabric CA server的配置文件`_ 章节.

`Back to Top`_

.. _client:

Fabric CA 客户端
----------------

这一节讲述如何使用fabric-ca-client的命令。

首先，要先确定Fabric CA client的home目录，其决定顺序如下:

  - 命令行 -home 选项 
  - 环境变量 ``FABRIC_CA_CLIENT_HOME`` 
  - 环境变量 ``FABRIC_CA_HOME``
  - 环境变量 ``CA_CFG_PATH``
  - ``$HOME/.fabric-ca-client``


请将配置文件放home目录下后，完成以下过程。

enroll the bootstrap identity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

大吉注：
	CA的账号概念是：先注册identity，它带有一个enrollment id，然后可以enroll具体的账号，可以在csr里指定属性，张三，李四，王二麻子
    从创建超级管理员到注册用户过程如下：
	bootstrap identity即超级管理员identity，注册是在ca-server初始化时完成的（用-b 选项指定enrollment ID和密码）。
    client端配置好CSR，并enroll了超级管理员identity到home目录下的msp
	client去向CA register user的identity，CA认可client的msp
	client去向CA enroll 刚才user的msp。  

首以下根据需要自定义client home目录下配置文件中的CSR部分，其中``csr.cn``必须设置为bootstrap identity的enrollment ID。
用fabric-ca-client gencsr -csr.cn admin 生成的客户端配置文件中的CSR默认值如下：

.. code:: yaml

    csr:
      cn: <<enrollment ID>>
      key:
        algo: ecdsa
        size: 256
      names:
        - C: US
          ST: North Carolina
          L:
          O: Hyperledger Fabric
          OU: Fabric CA
      hosts:
       - <<hostname of the fabric-ca-client>>
      ca:
        pathlen:
        pathlenzero:
        expiry:

请见 `CSR fields <#csr-fields>`__ 查看各配置项的描述.

然后运行 ``fabric-ca-client enroll`` 命令去enroll一个identity。例如,
以下命令会enroll一个ID是 **admin** 密码是 **adminpw** 的identity，
其调用的是运行在本地的监听7054端口的Fabric CA 服务器。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client enroll -u http://admin:adminpw@localhost:7054

enroll命令会生成一份enrollment 证书 (ECert), 以及对应的私钥文件和CA根证书 PEM 文件
，保存在Fabric CA client的 ``msp`` 子目录下。
提示信息里会告诉你保存到哪的目录下了。

注册一个新身份
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

发起注册请求的身份必须是已经登记（enroll）过的，同时也必须有权限去注册要注册的相应类型的身份。

在register期间，CA server会做两个授权检查:

 1. 调用者要register的身份必须是其“hf.Registrar.Roles”属性中所指明的身份中的一个。
    例如调用者的“hf.Registrar.Roles”属性值为“peer,app,user”，
	那么他不能register orderer类型的identity。

	
 2. 调用者identity的从属关系必须等于要register时候的从属关系的前缀。
    例如，一个调用者的从属关系是“a.b”，
	那么他可以register一个拥有”a.b.c”的identity，
	但不能是“a.c”。

下文的命令用admin identity去register一个新的identity，他的enrollment id是admin2，
类型是user，从属关系是org1.department1，hf.Revoker属性的值为true，foo属性的值为bar。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client register --id.name admin2 --id.type user --id.affiliation org1.department1 --id.attrs 'hf.Revoker=true,foo=bar'

CA server会返回一个密码，用于这个identity去enroll。
也允许一个管理员去register一个identity，
然后将这个identity对应的enrollment ID和密码给其他人去enroll。

多属性配置如下，可以用逗号隔开，如果属性里有逗号可以双引号括起来。

.. code:: bash

    fabric-ca-client register -d --id.name admin2 --id.type user --id.affiliation org1.department1 --id.attrs '"hf.Registrar.Roles=peer,user",hf.Revoker=true'

或

.. code:: bash

    fabric-ca-client register -d --id.name admin2 --id.type user --id.affiliation org1.department1 --id.attrs '"hf.Registrar.Roles=peer,user"' --id.attrs hf.Revoker=true

你也可以设置一个默认配置:

.. code:: yaml

    id:
      name:
      type: user
      affiliation: org1.department1
      maxenrollments: -1
      attributes:
        - name: hf.Revoker
          value: true
        - name: anotherAttrName
          value: anotherAttrValue

下面这条命令只指定了enrollment ID为
"admin3" 其余的属性都来自配置文件，如 type: "user", affiliation: "org1.department1",
及两个属性: "hf.Revoker" and "anotherAttrName".

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client register --id.name admin3

多属性需要像上面的配置文件那样提供属性名和属性值

如果设置 `maxenrollments` 为 0 或者不设置则其默认值为 CA的 最大 enrollment 值。
这个注册用户的最大enroll值不能超过CA的最大enroll值，
比如CA的最大值设置为是5，则所有注册的身份只能小于等于 5, 而且也不能设置为 -1 (无限enroll).

下面我们注册一个 **peer1** 用户，注意这里我们指定了密码，而不是让命令帮我们生成一个默认密码。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client register --id.name peer1 --id.type peer --id.affiliation org1.department1 --id.secret peer1pw

Enroll一个 Peer 身份
~~~~~~~~~~~~~~~~~~~~~~~~~

注册好身份后就可以enroll，enroll需要使用刚才注册的enrollmentID和密码(比如上节例子里的 *password*
).  这个enroll和enroll bootstrap身份有点像，只不过我们这里还用到了 "-M" 选项
用于指定生成 MSP (Membership Service Provider) 目录结构。

以下是enroll一个 peer1。
确保 "-M" 指定的目录为你的
peer的 MSP 目录， 要与peer的core.yaml文件里设置的
'mspConfigPath' 值要保持一致。
你也可以设置 FABRIC_CA_CLIENT_HOME 为你的 peer的home目录。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/peer1
    fabric-ca-client enroll -u http://peer1:peer1pw@localhost:7054 -M $FABRIC_CA_CLIENT_HOME/msp

enroll一个orderer也类似，只不过-M指定的是orderer.yaml里的 'LocalMSPDir' 。

从另一个CA服务器上获取一个 CA 证书链
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一般地，MSP文件夹中的证书文件夹必须包含其他证书认证中心的证书认证链，来代表这个peer是可信的。
以下命令启动了另一个CA server，这个代表完全分开的一个根信任，并且被区块链中的不同成员管理。

.. code:: bash

    export FABRIC_CA_SERVER_HOME=$HOME/ca2
    fabric-ca-server start -b admin:ca2pw -p 7055 -n CA2

以下命令将安装CA2的证书链到peer1的MSP文件夹中：

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/peer1
    fabric-ca-client getcacert -u http://localhost:7055 -M $FABRIC_CA_CLIENT_HOME/msp

重新enroll一个身份
~~~~~~~~~~~~~~~~~~~~~~~

假设你的证书到期了，就需要用以下命令重新enroll一份了

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/peer1
    fabric-ca-client reenroll

撤销一个证书或一个身份
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

撤销一个identity会撤销他的所有证书，并阻止他再得到新的证书，撤销一个证书只是使一个证书无效。
撤销者的从属关系是orgs.org1可以撤销从属关系是orgs.org1和orgs.org1.department1的identity，但不能撤销orgs.org1的identity。

命令如下：

.. code:: bash

    fabric-ca-client revoke -e <enrollment_id> -r <reason>

 ``-r`` 标志可以有如下选择：

  1. unspecified
  2. keycompromise
  3. cacompromise
  4. affiliationchange
  5. superseded
  6. cessationofoperation
  7. certificatehold
  8. removefromcrl
  9. privilegewithdrawn
  10. aacompromise


例子：bootstrap admin这个超级用户可以撤销**peer1**这个身份

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client revoke -e peer1

一个属于某个identity的证书可以通过指定他的AKI（Authority Key Identifier）和序列号来撤销:

.. code:: bash

    fabric-ca-client revoke -a xxx -s yyy -r <reason>

例如，可以用openssl 命令来得到AKI和序列号，然后将他们传入revoke命令来撤销证书：

.. code:: bash

   serial=$(openssl x509 -in userecert.pem -serial -noout | cut -d "=" -f 2)
   aki=$(openssl x509 -in userecert.pem -text | awk '/keyid/ {gsub(/ *keyid:|:/,"",$1);print tolower($0)}')
   fabric-ca-client revoke -s $serial -a $aki -r affiliationchange

使用 TLS
~~~~~~~~~~~~

这一节讲如何给 Fabric CA client配置TLS

以下是 ``fabric-ca-client-config.yaml``的内容

.. code:: yaml

    tls:
      # Enable TLS (default: false)
      enabled: true
      certfiles:
        - root.pem
      client:
        certfile: tls_client-cert.pem
        keyfile: tls_client-key.pem

**certfiles**选项指定被客户端信任的根证书，即CA的根证书，在CA 服务器的home 目录下的ca-cert.pem里。

**client**选项只有在server中配置了相同的TLS配置才用得到（大吉注：即双向认证）。

联系指定的 CA 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果服务器上跑了多个CA，如果未指定CA名，则将会访问到fabric-ca 服务上的默认CA。
CA名可以按如下方式指定:

.. code:: bash

    fabric-ca-client enroll -u http://admin:adminpw@localhost:7054 --caname <caname>

`Back to Top`_

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
