
What's the maximum size of each offset in a Kafka topic partition?

Fundamentally, the only maximum offset imposed by Kafka is that it has to be a 64-bit value.(A 64-bit signed integer. It has a minimum value of -9,223,372,036,854,775,808 and a maximum value of 9,223,372,036,854,775,807 (inclusive).)
