# Getting Started of IBM MQ on Red Hat Linux

## Create `mqm` group and `mqm` user

1. Create a group called `mqm`
   ```
   groupadd -g 500 mqm
   ```

1. Create a user called `mqm`.
   ```
   useradd -u 501 -g mqm -s /bin/bash -d /home/mqm -m mqm
   ```

1. Change password
   ```
   passwd mqm
   ```

1. Check created user/group
   ```
   id mqm
   uid=501(mqm) gid=500(mqm) groups=500(mqm)
   ```

## Create `mquser` group and `mquser` user

1. Create a group called `mqusers`
   ```
   groupadd -g 1000 mqusers
   ```

1. Create users called `alice` and `bob`
   ```
   useradd -u 1001 -g mqusers -s /bin/bash -d /home/alice -m alice
   useradd -u 1002 -g mqusers -s /bin/bash -d /home/bob -m bob 
   ```

1. Change password for users
   ```
   passwd alice
   passwd bob
   ```

1. Check create users/group
   ```
    id alice
    uid=1001(alice) gid=1000(mqusers) groups=1000(mqusers)
   
    id bob
    uid=1002(bob) gid=1000(mqusers) groups=1000(mqusers)
   ```

## Install MQ RPM components

1. Untar the installation files
   
   ```
   mkdir 9.4.2.0-IBM-MQ-LinuxX64
   tar xvf 9.4.2.0-IBM-MQ-LinuxX64.tar.gz -C 9.4.2.0-IBM-MQ-LinuxX64
   ```

1. Change directory
   ```
   cd  9.4.2.0-IBM-MQ-LinuxX64/MQServer
   ```

1. Accept license
   ```
   mqlicense.sh -accept
   ```

1. Obtain the IBM MQ public signing gpg key and install it into rpm.
   ```
   rpm --import ibm_mq_public.pgp
   ```

1. Install all RPM components
   ```
   rpm -Uvh MQSeries*.rpm
   ```
   or
   ```
   rpm --prefix /opt/customLocation -Uvh MQSeries*.rpm
   ```

1. Check installed components
   ```
   rpm -qa | grep MQ | xargs rpm -q --info
   ```

## Set environment and installation

1. Create a script file in `/usr/local/bin/set-mq-inst1`
   ```
   # Name: set-mq-inst1
   # Purpose: to setup the environment to run MQ
   . /opt/mqm/bin/setmqenv -n Installation1
   # Additional MQ directories for the PATH
   export PATH=$PATH:$MQ_INSTALLATION_PATH/java/bin:$MQ_INSTALLATION_PATH/samp/bin:$MQ_INSTALLATION_PATH/samp/jms/samples
   # Add local directory for running Java/JMS programs
   export CLASSPATH=$CLASSPATH:.
   # Display the full fix pack level
   dspmqver -f 2
   # end
   ```

2. Chmod a+x to the script
   ```
   chmod a+x /usr/local/bin/set-mq-inst1
   ```

3. Source the script
   ```
   . set-mq-inst1
   Version:     9.4.2.0
   ```

## Configure and tune Linux

1. Run `mqconfig` and ensure all values are `PASS`. See [IBM MQ configuration requirements](https://www.ibm.com/docs/en/ibm-mq/9.4?topic=linux-configuring-tuning-red-hat-enterprise) for details.
    ```
    mqconfig
    mqconfig: Analyzing Red Hat Enterprise Linux 9.4 (Plow) settings for IBM
            MQ V9.4

    System V Semaphores
    semmsl     (sem:1)  32000 semaphores                   IBM>=32           PASS
    semmns     (sem:2)  18 of 1024000000 semaphores (0%)    IBM>=4096         PASS
    semopm     (sem:3)  500 operations                     IBM>=32           PASS
    semmni     (sem:4)  7 of 32000 sets            (0%)    IBM>=128          PASS

    System V Shared Memory
    shmmax              18446744073692774399 bytes         IBM>=268435456    PASS
    shmmni              26 of 4096 sets            (0%)    IBM>=4096         PASS
    shmall              40150 of 18446744073692774399 pages (0%)    IBM>=2097152      PASS

    System Settings
    file-max            3264 of 9223372036854775807 files (0%)    IBM>=524288       PASS
    pid_max             716 of 4194304 processids  (0%)    IBM>=32768        PASS
    threads-max         716 of 125567 threads      (0%)    IBM>=32768        PASS

    Current User Limits (root)
    nofile       (-Hn)  10240 files                        IBM>=10240        PASS
    nofile       (-Sn)  10240 files                        IBM>=10240        PASS
    nproc        (-Hu)  126 of 62783 processes     (0%)    IBM>=4096         PASS
    nproc        (-Su)  126 of 62783 processes     (0%)    IBM>=4096         PASS   
    ```

2. Update the above values (if needed)

## Verify local server installation

1. Create queue mananger
   ```
   crtmqm QMA
   ```
2. Start queue manager
   ```
   strmqm QMA
   ```
3. Run MQ configuration
   ```
   runmqsc QMA
   ```
4. Define a local Q
   ```
   DEFINE QLOCAL(QUEUE1)
   ```
5. End MQ configuration
   ```
   END
   ```
6. Put messages into `QUEUE1` of `QMA`. Type some messages and enter a blank line to end input.
   ```
   amqsput QUEUE1 QMA
   ```
7. Get messages from `QUEUE1` of `QMA`.
   ```
   amqsget QUEUE1 QMA
   ```

## Verify client connection

1. Create queue mananger
   ```
   crtmqm QUEUE.MANAGER.1
   ```
   sr
2. Start queue manager
   ```
   strmqm QUEUE.MANAGER.1
   ```

3. Run MQ configuration
   ```
   runmqsc QUEUE.MANAGER.1
   ```

4. Configure the following
   ```
   DEFINE QLOCAL(QUEUE1)
   SET AUTHREC PROFILE(QUEUE1) OBJTYPE(QUEUE) PRINCIPAL('alice') AUTHADD(INQ,BROWSE,PUT,GET)
   DEFINE QLOCAL(MQSOURCE.STATE.Q)
   SET AUTHREC PROFILE(MQSINK.STATE.Q) OBJTYPE(QUEUE) PRINCIPAL('alice') AUTHADD(INQ,BROWSE,PUT,GET)
   DEFINE QLOCAL(MQSOURCE.STATE.Q)
   SET AUTHRECc PROFILE(TO.KAFKA.Q) OBJTYPE(QUEUE) PRINCIPAL('alice') AUTHADD(INQ,BROWSE,PUT,GET)
   SET AUTHREC OBJTYPE(QMGR) PRINCIPAL('alice') AUTHADD(CONNECT,INQ)
   DEFINE CHANNEL (CHANNEL1) CHLTYPE (SVRCONN) TRPTYPE (TCP)
   SET CHLAUTH(CHANNEL1) TYPE(ADDRESSMAP) ADDRESS('*') MCAUSER('alice')
   DEFINE LISTENER (LISTENER1) TRPTYPE (TCP) CONTROL (QMGR) PORT (1414)
   START LISTENER (LISTENER1)
   ```

4. Setup environment variable for client
   ```
   export MQSERVER='CHANNEL1/TCP/127.0.0.1(1414)'
   ```

5. Put message via client connection
   ```
   amqsputc QUEUE1 QUEUE.MANAGER.1
   ```
   or
   ```
   amqsputc QUEUE1 QUEUE.MANAGER.1 < test.xml
   ```

6. Get message via client connection
   ```
   amqsgetc QUEUE1 QUEUE.MANAGER.1
   ```

7. Start web console
   ```
   strmqweb
   ```
   