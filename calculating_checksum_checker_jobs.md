# How to calculate the size and frequencey of Islandora Checksum Checker jobs

* Goal is to spread checksum verification out as evenly as possible to reduce load. 
* Start with your preferred verifcation cycle: 1 month, 2 months, 3 months, 6 months, etc. Then with the number of objects in your repo: 10,000, 100,000, 500,000, 1,000,000, etc. One you have these two numbers, you can get the number of objects to verify per month. Once you have that, you can then calculate you need to calculate per day. Then, you can determine how many per job, using 1 job per hour as a start. For example:
  * 100,000 objects over 3 months = 33,333 per month = 1,389 per day = 58 per hour.
  * 500,000 objects over 3 months = 166,666 per month = 6,945 per day = 290 per hour.
  * 100,000 objects over 6 months = 16,666 per month = 695 per day = 28 per hour.
  * 500,000 objects over 6 months = 83,333 per month = 3,472 per day = 145 per hour.

* Relationship of datastream size to response time to verify checksum via Fedora's REST API
  * Querying Fedora's REST interface to verify checksums on datastreams of various sizes, e.g., `curl -s -w "%{time_total}\n" -u "fedoraAdmin:fedoraAdmin" -o /dev/null "http://localhost:8080/fedora/objects/islandora:29/datastreams/s1?format=xml&validateChecksum=true"`, produces the following rough data:
    * 1 MB: 0.030 seconds
    * 10 MB: 0.070 seconds
    * 50 MB: 0.270 seconds
    * 125 MB: 0.700 seconds
    * 250 MB: 1.300 seconds
    * 500 MB: 2.600 seconds
    * 1000 MB: 5.800 seconds
  * So it would appear that the lenght of time it takes to validate a checksum is proportionate to the size of the datastream.
* Number of datastreams: OBJ only, others, all?
