# PHP backend for resumable.js

This is a fork from [dilab/resumable.php](https://github.com/dilab/resumable.php) with changes to make it compatible with the [Laravel Framework](https://github.com/laravel/laravel)

It is currently a work in progress and it is currently developed with Laravel versions 5.1 and 5.2 in mind.

## Installation

To install, use composer:

``` composer require black-bits/resumable.js-laravel-backend ```


## How to use
**app/Http/routes.php**

```
<?php

// resumable.js routes
Route::get ('resumable/upload', 'UploadController@resumableUpload');
Route::post('resumable/upload', 'UploadController@resumableUpload');
```


**app/Http/Controllers/UploadController.php**

```
<?php
namespace App\Http\Controllers;

use Illuminate\Http\Response;

use Illuminate\Support\Facades\File;

use Dilab\Network\SimpleRequest;
use Dilab\Network\SimpleResponse;
use Dilab\Resumable;

class UploadController extends Controller
{
    /**
     * Handles resumeable uploads via resumable.js
     * 
     * @return Response
     */
    public function resumableUpload()
    {
        $tmpPath    = storage_path().'/tmp';
        $uploadPath = storage_path().'/uploads';
        if(!File::exists($tmpPath)) {
            File::makeDirectory($tmpPath, $mode = 0777, true, true);
        }

        if(!File::exists($uploadPath)) {
            File::makeDirectory($uploadPath, $mode = 0777, true, true);
        }

        $simpleRequest              = new SimpleRequest();
        $simpleResponse             = new SimpleResponse();

        $resumable                  = new Resumable($simpleRequest, $simpleResponse);
        $resumable->tempFolder      = $tmpPath;
        $resumable->uploadFolder    = $uploadPath;


        $result = $resumable->process();
        
        switch($result) {
            case 200:
                return response([
                    'message' => 'OK',
                ], 200);
                break;
            case 201:
                // mark uploaded file as complete etc
                return response([
                    'message' => 'OK',
                ], 200);
                break;
            case 204:
                return response([
                    'message' => 'Chunk not found',
                ], 204);
                break;
            default:
                return response([
                    'message' => 'An error occurred',
                ], 404);
        }
    }
}
```



## Testing
```
$ ./vendor/bin/phpunit
```

## Improvement 

it is not good for paralleled uploading in fast network. 
1. it reports error on make foldes
2. sometime it report error on createFileFromChunks

## changes

```
public function tmpChunkDir($identifier)
    {
        $tmpChunkDir = $this->tempFolder . DIRECTORY_SEPARATOR . $identifier;

        if (!file_exists($tmpChunkDir)) {
            if (!Cache::tags('resumable')->has('exist_' . $tmpChunkDir)) {
                sleep(rand(1,5));
                if (!Cache::tags('resumable')->has('exist_' . $tmpChunkDir)) {
                    Cache::tags('resumable')->put('exist_' . $tmpChunkDir, '1', 5);
                    //_l('re', [$identifier, $tmpChunkDir], 'debug', 1);
                    mkdir($tmpChunkDir);
                }
            }
        }
        return $tmpChunkDir;
    }
    
    public function handleChunk()
    {
        $file = $this->request->file();
        $identifier = $this->resumableParam('identifier');
        $filename = $this->resumableParam('filename');
        $chunkNumber = $this->resumableParam('chunkNumber');
        $chunkSize = $this->resumableParam('chunkSize');
        $totalSize = $this->resumableParam('totalSize');

        if (!$this->isChunkUploaded($identifier, $filename, $chunkNumber)) {
            $chunkFile = $this->tmpChunkDir($identifier) . DIRECTORY_SEPARATOR . $this->tmpChunkFilename($filename, $chunkNumber);
            $this->moveUploadedFile($file['tmp_name'], $chunkFile);
        }

        if ($this->isFileUploadComplete($filename, $identifier, $chunkSize, $totalSize)) {
            if (!Cache::tags('resumable')->has('complete_' . $identifier)) {
                sleep(rand(1,5));
                if (!Cache::tags('resumable')->has('complete_' . $identifier)) {
                    Cache::tags('resumable')->put('complete_' . $identifier, '1', 5);
                    //_l('re-compelete', [$filename, $identifier, $chunkSize, $totalSize], 'debug', 1);
                    $this->createFileAndDeleteTmp($identifier, $filename);
                }
            }
            return $this->response->header(201);

        }

        return $this->response->header(200);
    }

=
```

