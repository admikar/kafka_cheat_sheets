#to maintenance
sudo puppet agent --disable {TASK}
sudo /usr/bin/supervisorctl status
sudo /usr/bin/supervisorctl stop kafka-broker
sudowrapper.pl rm /etc/supervisord.d/kafka-broker.conf
sudowrapper.pl rm /etc/supervisord.d/crashmail_kafka-broker.conf
sudo /usr/bin/supervisorctl update
sudo /usr/bin/supervisorctl status

#back to prod
sudo puppet agent --enable
sudo puppet agent -t
sudo /usr/bin/supervisorctl status
