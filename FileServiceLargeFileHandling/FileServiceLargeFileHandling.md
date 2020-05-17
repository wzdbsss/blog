# File Service - Large File Handling

File service is currently capable of handling large files (up to 5GB) and write them directly to AWS S3 by using a Java InputStream. By using a Java InputStream the binary data is streamed directly to S3 and not held in memory where large files will quickly exhaust the available memory. The binary data input stream is pulled from the HttpServletRequest and passed to the file service as seen in line 26 `request.getInputStream()` in the code below.  Also note that the content length of the stream is pulled in line 27. The first check in file service compares the content length to the max file size limit in order to validate the stream before streaming all of its contents. 

``` java
@Override
@PreAuthorize("#oauth2.hasScope('iot.fil.w')")
public ResponseEntity<Void> putFile(
        HttpServletRequest request,
        @ApiParam(value = "unique identifier of the entity", required = true)
                @PathVariable("entityId")
                String entityId,
        @ApiParam(value = "type of the file") @RequestHeader(value = "type", required = false)
                String type,
        @ApiParam(value = "ETag of the latest version for optimistic locking")
                @RequestHeader(value = "If-Match", required = false)
                Integer ifMatch,
        @ApiParam(value = "file timestamp")
                @RequestHeader(value = "timestamp", required = false)
                String timestamp,
        @ApiParam(value = "description of the file")
                @RequestHeader(value = "description", required = false)
                String description)
        throws IOException {
 
    String filepath = FileServiceUtil.getFilePath(request, entityId);
    FileData fd = fileService.getFileData(entityId, filepath);
    try {
        MultiValueMap<String, String> headers =
                fileService.putFile(
                        request.getInputStream(),
                        request.getContentLengthLong(),
                        entityId,
                        filepath,
                        ifMatch,
                        timestamp,
                        request.getHeader(HttpHeaders.AUTHORIZATION),
                        description,
                        type);
        if (fd != null) {
            return new ResponseEntity<Void>(headers, HttpStatus.NO_CONTENT); //Update
        } else {
            return new ResponseEntity<Void>(headers, HttpStatus.CREATED); //Create
        }
    } catch (ConflictingETagException ex) {
        return new ResponseEntity<Void>(HttpStatus.CONFLICT);
    }
}
```
The Amazon S3 API provides a putObject method which takes an InputStream and writes the binary data directly to S3 as seen in the following code. 
``` java
private PutObjectResult saveToS3(
        String bucketName,
        String entityId,
        String filepath,
        TenantVo tenantVo,
        InputStream inputStream,
        ObjectMetadata inputMetadata) {
    PutObjectResult objectResult;
    try {
        objectResult =
                s3.putObject(
                        bucketName,
                        getS3Key(entityId, filepath, tenantVo),
                        inputStream,
                        inputMetadata);
    } catch (AmazonS3Exception ex) {
        LOGGER.info("Unable to save file to S3.");
        throw new S3AccessDeniedException(ex);
    }
    return objectResult;
}
```
When retrieving files from S3, file service uses the the getObject method of S3Object that contains an S3ObjectInputStream. The S3ObjectInputStream extends the standard java.io.InputStream and is handled the same way as seen in the code snippit below
``` java
@Override
public InputStream retrieveFile(String entityId, String filepath, String accessToken) {
    LOGGER.info("Retrieving file " + filepath);
 
    String bucketName = null;
    try {
        //... validations ...
 
        //Get the bucket name from the tenant
        bucketName = tenantVo.getResources().get(FileServiceUtil.BUCKET_NAME);
        if (bucketName != null) {
            S3Object s3Object =
                    s3.getObject(bucketName, getS3Key(entityId, filepath, tenantVo));
            return s3Object.getObjectContent();
        } else {
            throw new TenantBucketNotFoundException(null);
        }
 
    } catch (AmazonServiceException ex) {
        //... exception handling ...
        }
        throw ex;
    } catch (RestClientException ex) {
        LOGGER.info(ex.getLocalizedMessage());
        throw ex;
    }
}
```
The Controller can then return the InputStream in the ResponseEntity without holding the data in memory as seen below. 
``` java
@Override
@PreAuthorize("#oauth2.hasScope('iot.fil.r')")
public ResponseEntity<InputStreamResource> getFile(
        HttpServletRequest request,
        @ApiParam(value = "Id to instance of entity", required = true) @PathVariable("entityId")
                String entityId,
        @ApiParam(value = "ETag of the latest version")
                @RequestHeader(value = "If-None-Match", required = false)
                Integer ifNoneMatch) {
 
    String filepath = FileServiceUtil.getFilePath(request, entityId);
    InputStream inputStream =
            fileService.retrieveFile(
                    entityId, filepath, request.getHeader(HttpHeaders.AUTHORIZATION));
 
    InputStreamResource inputStreamResource = new InputStreamResource(inputStream);
    return new ResponseEntity<InputStreamResource>(inputStreamResource, HttpStatus.OK);
}
```
