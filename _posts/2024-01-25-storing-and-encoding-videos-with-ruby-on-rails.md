---
title: "Как хранить и кодировать видео посредством Ruby on Rails"
tags: [RUBY ON RAILS]
style: border
color: danger
description: В статье мы рассмотрим простой способ, который подходит как для небольших, так и для средних приложений
---
В связи с тем, что все больше пользователей предпочитают видеоконтент изображениям, пользовательским приложениям чаще
приходится загружать и воспроизводить видео. 

Необходимость работать с видео в условиях ненадежного интернет-соединения требует, чтобы качество его обработки при 
загрузке и передаче (скачивании другими пользователями) соответствовало доступной пропускной способности. 

Для достижения этих целей был разработан ряд решений. Они включают в себя такие SaaS-предложения, как [Mux](https://mux.com/) 
и [AWS Elemental](https://www.elemental.com/).

В статье мы рассмотрим простой способ, который подходит как для небольших, так и для средних приложений, 
интегрированных с пользовательским бэкендом, но использующих инфраструктуру AWS для обработки и передачи видео. 

**Разрабатываемый подход предусматривает применение 3 ключевых компонентов.**

AWS S3 с [Transfer Acceleration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/transfer-acceleration.html). Обеспечивает масштабируемое местоположение с быстрым доступом для загрузки и хранения
файлов. 
AWS Lambda. Предназначен для перекодировки видеофайлов в любое количество желаемых форматов и требуемое качество. 
Пользовательский бэкенд. Бэкенд Ruby on Rails подходит для любого языка программирования. Необходим для хранения 
метаданных о видео и предоставления их клиентам.
Процесс загрузок следующий: запрашиваем подписанный URL-адрес из бэкенда Rails, после чего задействуем предоставленный 
URL для прямой загрузки видеофайла в S3. 

После загрузки с помощью функциональности оповещения о событиях [S3 Event Notification](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html) запускаем Lambda для обработки 
видео. Полученные метаданные, такие как ключ S3, сохраняются в пользовательском бэкенде. Итоговый поток данных при
загрузке нового видеофайла выглядит следующим образом: 
## Реализация 

### Часть 1. Видеозапись (видеообъект) в бэкенде 
Для отслеживания загруженных видео создадим в бэкенде модель с такими важными свойствами, как: 

- строка key для хранения ключа S3 исходного загруженного файла и аналогичного атрибута для каждого востребованного
производного видеофайла;
- low_res_key для хранения ключа S3 к производному видеофайлу с низким разрешением от исходного файла,
обработанного Lambda. 
**В Ruby on Rails требуемая миграция выглядит так:**
```ruby
  class CreateVideo < ActiveRecord::Migration[6.1]
  def change
    create_table :videos, id: :uuid do |t|
      t.string :key, unique: true, index: true
      t.string :low_res_key, unique: true, index: true
      t.references :user, foreign_key: true, type: :uuid
      t.timestamps
    end
  end
end
```
Соответствующая модель видео обязательно включает 2 метода.

1. Метод для назначения уникального ключа новому видео (можно использовать id видеообъекта). 
2. Метод для извлечения криптографически подписанного URL-адреса загрузки в корзину AWS S3. 
В результате получаем такую модель: 

```ruby
class Video < ApplicationRecord
  UPLOAD_BUCKET = "medium-article-#{Rails.env}-videos"
  belongs_to :user, optional: true
  before_save :assign_random_key
  
  def original_url
    "https://#{ENV['CLOUDFRONT_DOMAIN']}/#{key}"
  end
  def low_res_url
    return unless low_res_key
    
    "https://#{ENV['CLOUDFRONT_DOMAIN']}/#{low_res_key}"
  end
  def upload_url
    presigner = Aws::S3::Presigner.new
    presigner.presigned_url(
      :put_object,
      bucket: UPLOAD_BUCKET,
      key: key,
      use_accelerate_endpoint: true,
      expires_in: 900
    )
  end
  private
  
  def assign_random_key
    return if key.present?
    
    self.key = "uploads/#{user_id}/#{SecureRandom.uuid}"
  end
end
```
### Часть 2. Создание и получение видеозаписи из бэкенда 
Прежде всего, найдем способ создавать видеозаписи и извлекать необходимую информацию для загрузки видео. 

Для этого потребуются несколько методов контроллера.

1. Метод POST. Необходим для создания новой видеозаписи. Обратите внимание, что при создании видеообъекту автоматически назначается ключ. Однако этот ключ не сразу указывает на существующий файл S3, поскольку фактический видеофайл может быть еще не загружен. 
2. Метод GET. Предназначен для получения видеозаписи. 
3. Метод PUT. Лямбда, создающая производные видеофайлы, с помощью данного метода передает на сервер ключи для сгенерированных производных файлов. 

```ruby
class Api::VideosController < Api::ApplicationController
  skip_before_action :authenticate_user, only: %i[derivative]
  # POST /api/videos
  def create
    @video = current_user.videos.new
    if @video.save
      render json: VideoSerializer.render(@video, view: :with_upload_details), status: :ok
    else
      render json: @video.errors, status: :bad_request
    end
  end
  # GET /api/videos
  def show
    @video = current_user.videos.find(params[:id])
    render json: VideoSerializer.render(@video), status: :ok
  end
  # PUT /api/videos/derivative
  def derivative
    # Разрешает только запросы от внутренних сервисов 
    return head :forbidden unless ENV['LAMBDA_SHARED_SECRET'] == request.headers['Authorization']
    @video = Video.find_by(key: params[:key])
      
    if @video.update(safe_derivative_params)
      head :ok
    else
      render json: @video.errors, status: :bad_request
    end
  end
  private
  def safe_derivative_params
    params.permit(:low_res_key)
  end
end
```
### Часть 3. Обработка загруженного видео 
Воспользуемся S3 Event Notification и автоматически запустим выполнение Lambda для обработки видеофайла, загруженного в S3. 

Для объявления и развертывания Lambda и S3 будем задействовать [AWS Serverless Application Model](https://aws.amazon.com/serverless/sam/) (SAM). Однако можно применить ниже представленный код и вручную создать Lambda и S3 из консоли AWS.

Создаем следующую структуру каталога и файла: 
```ruby
video-processor/
├── cmd/
│   └── deploy.sh
├── src/
│   ├── s3-util.js
│   ├── child-process-promise.js
│   └── index.js
├── .gitignore 
└── template.yaml
```
child-process-promise.js определяет вспомогательную функцию, которая запускает новый процесс внутри промиса. Воспользуемся им для вызова FFMPEG в основном коде Lambda:
```js
const childProcess = require('child_process'),
	spawnPromise = function (command, argsarray, envOptions) {
		return new Promise((resolve, reject) => {
			console.log('executing', command, argsarray.join(' '));
      
			const childProc = childProcess.spawn(command, argsarray, envOptions || {env: process.env, cwd: process.cwd()})
      const resultBuffers = [];
      
			childProc.stdout.on('data', buffer => {
				console.log(buffer.toString());
				resultBuffers.push(buffer);
			});
      
			childProc.stderr.on('data', buffer => console.error(buffer.toString()));
      
			childProc.on('exit', (code, signal) => {
				console.log(`${command} completed with ${code}:${signal}`);
				if (code || signal) {
					reject(`${command} failed with ${code || signal}`);
				} else {
					resolve(Buffer.concat(resultBuffers).toString().trim());
				}
			});
		});
	};

module.exports = {
	spawn: spawnPromise
};
```
s3-util.js определяет вспомогательный метод для скачивания видеофайлов из S3.
```js
/*global module, require, Promise, console */

const aws = require("aws-sdk");
const fs = require("fs");
const s3 = new aws.S3();

const downloadFileFromS3 = function (bucket, fileKey, filePath) {
  "use strict";
  console.log("Downloading", bucket, fileKey, filePath);
  return new Promise(function (resolve, reject) {
    const file = fs.createWriteStream(filePath),
      stream = s3
        .getObject({
          Bucket: bucket,
          Key: fileKey,
        })
        .createReadStream();
    stream.on("error", reject);
    file.on("error", reject);
    file.on("finish", function () {
      console.log("downloaded", bucket, fileKey);
      resolve(filePath);
    });
    stream.pipe(file);
  });
};

module.exports = {
  downloadFileFromS3: downloadFileFromS3,
};
```
Полный вариант кода для Lambda представлен ниже в index.js. Его можно разделить на 5 частей. 

1. Скачивание видеофайла из S3 в рабочий каталог Lambda. Файлы клиентским приложением загружаются в /uploads.  
2. Обработка скаченного видеофайла посредством ffmpeg.
3. Загрузка нового обработанного видеофайла в S3. Lambda загружает производные файлы в /processed.
4. Информирование сервера о готовности нового файла. 
5. Удаление файла из Lambda. Необходимость этого шага объясняется тем, что пространство хранения Lambda может совместно использоваться разными процессами выполнения. Если вы не удаляете файлы, они накапливаются и заполняют доступное пространство. 

```js
const s3Util = require("./s3-util");
const childProcessPromise = require("./child-process-promise");
const path = require("path");
const os = require("os");
const fs = require("fs");
const https = require("https");

const OUTPUT_BUCKET = process.env.OUTPUT_BUCKET;
const VIDEO_MIME_TYPE = process.env.VIDEO_MIME_TYPE;
const LAMBDA_SHARED_SECRET = process.env.LAMBDA_SHARED_SECRET;

exports.handler = async (eventObject, context) => {
  const eventRecord = eventObject.Records && eventObject.Records[0];
  const inputBucket = eventRecord.s3.bucket.name;
  const key = eventRecord.s3.object.key;
  const id = context.awsRequestId;
  const workdir = os.tmpdir();

  // Получение имени файла без пути 
  const filename = path.basename(key); // /path/1/.../n/filename

  // Имена файлов после их размещения в выходной корзине S3  
  const lowResKey = "processed/lowRes/" + filename;

  // Имена файлов, пока они находятся в Lambda перед загрузкой в выходную корзину 
  const inputFile = path.join(workdir, id + path.extname(key));
  const lowResOutputFile = path.join(workdir, "lowRes-" + id + ".mp4");

  console.log("Download file from S3...");
  await s3Util.downloadFileFromS3(inputBucket, key, inputFile);

  // lowRes
  console.log("Generate lowRes optimized (plays on Chrome) video file...");
  await childProcessPromise.spawn(
    "/opt/bin/ffmpeg",
    [
      "-loglevel",
      "error",
      "-y",
      "-i",
      inputFile,
      "-movflags",
      "faststart",
      "-vf",
      "scale=480:-2",
      lowResOutputFile,
    ],
    { env: process.env, cwd: "./" }
  );

  console.log("Upload lowRes file...");
  await s3Util.uploadFileToS3(
    OUTPUT_BUCKET,
    lowResKey,
    lowResOutputFile,
    VIDEO_MIME_TYPE
  );

  // Обратный вызов к API для оповещения о завершении обработки этого файла 
  console.log("Informing server lowRes derivative is ready...");
  await informServerOfCompletion({ key: key, low_res_key: lowResKey });

  // Очистка оставшихся артефактов 
  console.log(
    "Delete unused files to avoid running out of space on future runs..."
  );
  fs.unlinkSync(inputFile); // Original
  fs.unlinkSync(lowResOutputFile);

  console.log(`Signaling post process complete for ${key}`);
};

// Вспомогательный метод 
const informServerOfCompletion = async (data) => {
  data = JSON.stringify(data);

  const options = {
    hostname: process.env.API_HOSTNAME,
    port: 443,
    path: "/api/videos/derivative",
    method: "PUT",
    headers: {
      "Content-Type": "application/json",
      Accept: "application/json",
      "Content-Length": data.length,
      "User-Agent": "Lambda-PreviewGenerator",
      Authorization: LAMBDA_SHARED_SECRET,
    },
  };

  console.log("Callback to server...", data);
  await new Promise((resolve) => {
    const req = https.request(options, (res) => {
      console.log(`/api/videos/derivative statusCode: ${res.statusCode}`);

      res.on("data", (d) => {
        process.stdout.write(d);
        resolve();
      });
    });

    req.on("error", (error) => {
      console.log(`/api/videos/derivative failed ${error}`);
      resolve();
    });

    req.write(data);
    req.end();
  });
};
```
AWS SAM позволяет определить функцию Lambda и связанные с ней ресурсы в файле YAML, а также автоматически развернуть и обновить ее с помощью инструментов командной строки, предоставляемых AWS. 

Файл template.yaml объявляет используемые ресурсы и триггер по созданию файла в каталоге /uploads (также называемый prefix).
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM project for Medium article about uploading and processing videos

Parameters:
  EnvironmentValue:
    AllowedValues:
      - "staging"
      - "production"
    Default: "staging"
    Description: "What environment is this?"
    Type: String

Mappings:
  Environments:
    staging:
      APIHOSTNAME: api-staging.example.com.br
      LAMBDASHAREDSECRET: your-super-secret-shared-key
    production:
      APIHOSTNAME: api.example.com.br
      LAMBDASHAREDSECRET: your-super-secret-shared-key

Resources:
  VideoDerivativeGenerator:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/video_derivative_generator
      Handler: index.handler
      Runtime: nodejs12.x
      MemorySize: 3008
      Timeout: 600
      Environment:
        Variables:
          VIDEO_MIME_TYPE: video/mp4
          LAMBDA_SHARED_SECRET:
            !FindInMap [Environments, !Ref EnvironmentValue, LAMBDASHAREDSECRET]
          OUTPUT_BUCKET: !Sub "example-${EnvironmentValue}-videos"
          API_HOSTNAME:
            !FindInMap [Environments, !Ref EnvironmentValue, APIHOSTNAME]
      Policies:
        - AWSLambdaExecute # Управляемая политика 
        - Version: "2012-10-17" # Policy Document
          Statement:
            - Effect: Allow
              Action:
                - logs:CreateLogGroup
                - logs:CreateLogStream
                - logs:PutLogEvents
                - s3:*
              Resource: "*"
      Events:
        VideoUploaded:
          Type: S3
          Properties:
            Bucket:
              Ref: VideosBucket
            Events:
              - "s3:ObjectCreated:*"
            Filter:
              S3Key:
                Rules:
                  - Name: prefix
                    Value: uploads/
      Layers:
        - arn:aws:lambda:<your-aws-region>:<your-aws-account-id>:layer:ffmpeg:1

  VideosBucket:
    Type: "AWS::S3::Bucket"
    Properties:
      BucketName: !Sub "example-${EnvironmentValue}-videos"
      AccelerateConfiguration:
        AccelerationStatus: Enabled
```
## Заключение 
Мы рассмотрели процесс создания инфраструктуры, поддерживающей загрузку и обработку видеофайлов для мобильного приложения или его аналога. Система отлично масштабируется, поскольку задействует инфраструктуру AWS для решения сложных задач, таких как обработка видеофайлов. 

Обратим внимание на 2 момента.

1. Обработка видеофайлов на стороне сервера не исключает необходимости обработки файлов на стороне клиента. Дело в том, что исходные файлы могут быть слишком большими для загрузки в условиях нормального интернет-соединения. 
2. Допускается добавление дополнительных производных видеофайлов. Для этого нужно внести в код Lambda изменения, предусматривающие создание большего количества видеофайлов на основе требуемых спецификаций. Например, может потребоваться ограниченная по времени версия загруженных видеофайлов для предварительных просмотров. 
