---
title:	"Stream Files to Amazon S3"
date:	2015-04-20
---
For any SaaS platform it is common to use a 3rd party hosting service for uploading files and serving them through a CDN. Amazon S3 is a common choice.

Usually the file upload from the client side (say, AngluarJS) is sent to the server (node server running on GNU/Linux box) which is then forwarded to Amazon S3. This approach is inefficient because the file needs to be opened and "read" by the server and then forwarded to S3. The solution provided here 
<ol>
<li> Extracts all the meta data (file name, size, mime type) </li>
<li> Opens a File stream to the Amazon S3 bucket </li>
<li> And writes the file directly to it. </li>
</ol>

Snippet with comments :

{% highlight javascript %}
var express = require('express');
var router = express.Router();

var multer = require('multer'),	//for handling multipart/form-data
fs = require('fs'),
S3FS = require('s3fs'),	//abstraction over Amazon S3's SDK
s3fsImpl = new S3FS('your-bucket-here', {
       accessKeyId: 'Your-IAM-Access',
       secretAccessKey: 'Your-IAM-Secret'
   	});


// POST a new Path 
router.post('/', [ multer(), function(req, res, next) {
	var file = req.files.file;
	console.log(file);

/* Output:
{ 
	fieldname: 'file',
	originalname: 'ice-boxes.jpg',
	name: '2658a8f666e33ab1ec39dc8b7b20970b.jpg',
	encoding: '7bit',
	mimetype: 'image/jpeg',
	path: 'public/uploads/2658a8f666e33ab1ec39dc8b7b20970b.jpg',
	extension: 'jpg',
	size: 88076,
	truncated: false,
	buffer: null 
 }
*/

	//Create a file stream
   var stream = fs.createReadStream(file.path);	

   //writeFile calls putObject behind the scenes
   s3fsImpl.writeFile(file.name, stream).then(function () {	
        fs.unlink(file.path, function (err) {
            if (err) {
                console.error(err);
            }
        });
        res.status(200).end();
    });

}]);


module.exports = router;
{% endhighlight %}


I struggled with it for quite some time when I needed to implement this feature in one of my projects. I hope it will help the community in future.

Resources: [Multer](https://www.npmjs.com/package/multer), [S3FS](https://github.com/RiptideCloud/s3fs)