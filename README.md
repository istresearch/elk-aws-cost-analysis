# elk-aws-cost-analysis
Visualize your Amazon Web Services (AWS) costs using ElasticSearch, Logstash and Kibana (ELK) in Docker

# Required environment variables
The logstash.conf file depends on the following environment variables.  The values are not embedded in the file since that would expose them to the public via GitHub.  Create a ".env" file in the same directory as the docker-compose.yml file and docker-compose will automatically read the file.

    LOGSTASH_ES_USER=[your ElasticSearch cluster username]  
    LOGSTASH_ES_PWD=[your ElasticSearch cluster password]  
    ES_HOST=[your ElasticSearch cluster hostname:port]  

# Where do I get the billing data from?
AWS can be setup to generate detailed billing information based on your usage.  You may specify tags for each of your systems if you want to split your billing data into your own logical configuration.  See https://aws.amazon.com/answers/account-management/aws-tagging-strategies/ and https://aws.amazon.com/blogs/aws/new-aws-resource-tagging-api/.  You will need to configure your tags as cost allocation tags as well, see http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html.

Once you have configured your tags you will need to enable detailed billing file generation, see Detailed Billing Report with Resources and Tags in http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/billing-reports.html#other-reports.

I download the billing data from S3 using the AWS CLI tool using the sync command from a bucket called "mybilling" that I setup following the instructions in the links above:

    aws s3 sync s3://mybilling mybilling/.

# Starting ELK
Start up ElasticSearch and Kibana:

    docker-compose up -d elasticsearch kibana

Once ES and Kibana are up you can start feeding data into ES:

    docker-compose up logstash

Note that the billing file is expected to be in a file call "billing.csv".

# How do I actually use this?
I have an ES index running on a cluster all of the.  I happen to use ElasticCloud, but you can run it whereever you like.  A couple of times a day a cron job runs to sync the latest billing file from AWS to my server.  This cron job then deletes the previous index (why?  see below) and then rebuilds it from the latest billing file.

Here is the shell script that is called a couple of times a day from cron, the `aws-cost` directory is where I keep this project:

    cd ~/aws-cost
    aws s3 sync s3://mybilling mybilling/.
    # This needs to be improved, right now I update the next two lines each month to grab the right month's file.
    unzip mybilling/999999999999-aws-billing-detailed-line-items-with-resources-and-tags-2017-03.csv.zip
    mv 999999999999-aws-billing-detailed-line-items-with-resources-and-tags-2017-03.csv billing.csv
    curl -Xdelete "https://<your ES cluster>/billing-qcr"
    docker-compose up logstash
    docker-compose down

# Credit
Thanks go to my co-worker https://github.com/madisonb who got the stub of this up and running for me one afternoon.

# BIG FAT NOTE
*This is not working right now on Logstash 5.3 since Elastic broke the ability to use environment variables in logstash.conf files.  The workaround is to drop back to Logstash 5.1.1.*
https://github.com/elastic/logstash/issues/6696
