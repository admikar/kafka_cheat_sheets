For a rolling upgrade:

    Prepare for System Upgrade (p.1 and p.2)
    Prepare hiera and commit. Update server.properties on all brokers and add the following properties (version=old version, you can check from  page https://docs.confluent.io/current/installation/versions-interoperability.html).
        inter.broker.protocol.version={$CUR_VER}
        log.message.format.version={$CUR_VER}
    Run "puppet agent --disable update" on all brokers and MM(all or needed) switch to maintenance "touch ~/.maintenance_mode" onto host.
    Run pipleline the brokers one by one:
        Remove broker from cluster alias and wait correct resolve.
        Run "kafka-server-stop" (with monitoring server.log "INFO [KafkaServer id=#] shut down completed (kafka.server.KafkaServer)" and after shut down "sudo /usr/bin/supervisorctl stop kafka-broker")
        Do System Upgrade (p.3)
        Unlock puppet agent("puppet agent --enable") and run "puppet agent -t"
        Next "puppet agent --disable update" again
        Wait till all partitions will in ISR (kafka-topics --describe --under-replicated-partitions --zookeeper localhost:2181 must be empty)
        Return back broker to cluster alias and wait correct resolve.
    Prepare hiera and commit
        __wgdp::kafka::package_name
        __wgdp::schema_registry::package_name
        __wgdp::wg::packages_set::kafka_rest_utils::version
    Upgrade the brokers one at a time:
        shut down the broker(Remove broker from cluster alias and wait correct resolve.)
        update the code with avto restart it, Run "kafka-server-stop" (with monitoring server.log "INFO [KafkaServer id=#] shut down completed (kafka.server.KafkaServer)" and after shut down "sudo /usr/bin/supervisorctl stop kafka-broker")
        lock puppet agent, "puppet agent --disable update"
        Wait till all partitions will in ISR (kafka-topics --describe --under-replicated-partitions --zookeeper localhost:2181 must be empty)
    Once the entire cluster is upgraded, bump the protocol version by editing inter.broker.protocol.version and setting it to 1.0. Commit to hiera
        inter.broker.protocol.version={$NEW_VER}
    Restart the brokers one by one for the new protocol version to take effect. (as 4)
        log.message.format.version={$NEW_VER}
    Unlock puppet agent on all kafka brokers "puppet agent --enable" and "rm ~/.maintenance_mode" from MM.
