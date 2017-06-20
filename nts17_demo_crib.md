# Create AWS node-red instance

* Start from a folder containing your AWS certificate (`.pem` file).

* Launch a new instance Ubuntu 16 / t2.micro.

* Login using ssh instructions via connect button in EC2 platform.

* Install Node.JS 6 according to [these instructions](https://nodejs.org/en/download/package-manager/#debian-and-ubuntu-based-linux-distributions):

      curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
      sudo apt-get install -y nodejs

      sudo apt-get install -y build-essential 
      
* Install python2.7 and (configure it for use in builds)[https://github.com/nodejs/node-gyp]. 

      sudo apt-get install -y python2.7
      npm config set python /usr/bin/python2.7

* Install node-red globally:

      sudo npm install -g node-red
    
* Configure the firewall, opening ports 1880 and 8712.

* Run node red. Connect via the web address.

      node-red

* Install the following modules:

      node-red-contrib-dynamorse-core
      node-red-contrib-dynamorse-file-io
      node-red-contrib-dynamorse-http-io
    
* Copy over the BBC Countdown sound file.

      scp -i "sparkpunkeu.pem" ../media/sound/BBCNewsCountdown.wav ubuntu@...:.

Demo the plumbing of nodes between AWS and Geneva.

Finally, terminate the instance, destroying all of the work done above.

## Other details

PCAP file reading:

    Documents/media/examples/rtp-video-rfc4175-1080i50-sync.pcap 
    file:Documents/media/sdps/sdp_rfc4175_10bit_1080i50.sdp
    
