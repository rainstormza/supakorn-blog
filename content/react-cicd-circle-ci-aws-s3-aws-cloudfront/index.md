---
title: "เขียน React แล้วมาทำ CI/CD ด้วย Circle CI, AWS S3 และ AWS CloudFront กันเถอะ"
description: "ในปัจจุบันส่วนหนึ่งที่สำคัญของการพัฒนา Software คือ การ run test, build และ deploy อยู่เสมอ ซึ่งหากไม่ได้มีการทำ CI/CD เอาไว้ ก็จะต้องมาทำ Manual เองทั้งหมด ตั้งแต่การ run test ให้เสร็จ ต่อด้วย run…"
date: "2018-04-07T07:25:31.498Z"
categories: 
  - Circleci
  - React
  - Aws S3
  - Cloudfront
  - Web Development

published: true
canonical_link: https://blog.nextzy.me/%E0%B9%80%E0%B8%82%E0%B8%B5%E0%B8%A2%E0%B8%99-react-%E0%B9%81%E0%B8%A5%E0%B9%89%E0%B8%A7%E0%B8%A1%E0%B8%B2%E0%B8%97%E0%B8%B3-ci-cd-%E0%B8%94%E0%B9%89%E0%B8%A7%E0%B8%A2-circle-ci-aws-s3-%E0%B9%81%E0%B8%A5%E0%B8%B0-aws-cloudfront-%E0%B8%81%E0%B8%B1%E0%B8%99%E0%B9%80%E0%B8%96%E0%B8%AD%E0%B8%B0-6997aa08128a
redirect_from:
  - /เขียน-react-แล้วมาทำ-ci-cd-ด้วย-circle-ci-aws-s3-และ-aws-cloudfront-กันเถอะ-6997aa08128a
---

![ci / cd life cycle](./asset-1.png)

ในปัจจุบันส่วนหนึ่งที่สำคัญของการพัฒนา Software คือ การ run test, build และ deploy อยู่เสมอ ซึ่งหากไม่ได้มีการทำ CI/CD เอาไว้ ก็จะต้องมาทำ Manual เองทั้งหมด ตั้งแต่การ run test ให้เสร็จ ต่อด้วย run build ได้ไฟล์ build มา เอาไป deploy บน server รวมถึงเมื่อต้องการพัฒนา feature ใหม่ๆ หรือ มีการแก้ bug ต่างๆ ก็ต้องทำขั้นตอนทั้งหมดนี้ วนซ้ำไปเรื่อยๆ

จะดีกว่ามั้ยหากเราสามารถลดเวลาในการทำขั้นตอนซ้ำๆ เดิมๆ เหล่านี้ได้ และ สามารถระบุปัญหาหรือความผิดพลาดต่างๆ ที่เกิดขึ้นระหว่างการพัฒนา Software ได้อย่างรวดเร็ว ด้วยการทำ CI/CD ?

![](./asset-2.png)

---

#### TL;DR

-   สร้าง react app ขึ้นมาแล้วอัพขึ้น repo ของ github
-   Setting AWS S3 ไว้สำหรับ deploy
-   Setting AWS Cloudfront เพื่อทำ https และ ทำ CDN
-   Setting Circle CI ไว้สำหรับ subscribe repository และ trigger เมื่อมี commit เข้ามา ให้ทำการ test, build, deploy ลง AWS

---

#### Prerequisites

-   [Github Account](https://github.com)
-   [CircleCI Account](https://circleci.com)
-   [AWS Account](https://aws.amazon.com)
-   [AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

---

#### How it works

![ci / cd process](./asset-3.png)

-   เมื่อมีการ commit code ขึ้น repository ( เช่น github, bitbucket )
-   ตัว repository จะทำการ trigger (webhook) ไปบอกตัว ci ที่ subscribe repository นั้นๆอยู่
-   CI server ทำการรันคำสั่งตามที่เรากำหนดไว้ ( เช่น สั่งให้ run test, check syntax, build, deploy )
-   ถ้า process ไหน เกิดข้อผิดพลาด จะส่ง message มาบอกว่าผิดพลาดตรงส่วนไหน
-   developer ทำการ แก้บัคแล้ว commit code ขึ้นไปใหม่ ( กลับไปเริ่มต้นขั้นตอนแรก )

---

#### Getting Started

สร้าง repository ผ่าน github

![react-circleci repository](./asset-4.png)

สร้าง react project และทำการ เพิ่ม git เข้า project

```
$ npx create-react-app react-circleci
$ cd react-circleci

$ git init
$ git add .
$ git commit -m 'initial react circleci'
$ git remote add origin https://github.com/<user>/<repository-name>.git
$ git push -u origin master
```

( npx สามารถใช้ได้กับ npm version > 5.2 )

---

#### Setting up AWS

เข้าไปที่ [https://s3.console.aws.amazon.com/s3/](https://s3.console.aws.amazon.com/s3/) ( ต้องสร้าง account ก่อน ซึ่งตอนสร้าง account จำเป็นต้องใส่บัตรเครดิต แนะนำให้ใช้ WeCard ของ truemoney ) หลังจากนั้น login เข้ามา กดที่ปุ่ม **Create bucket** ใส่ชื่อ **Bucket name**: reactcircleci.com แล้วกด Next ไปเรื่อยๆ จนถึงขั้นตอนสุดท้าย กด Create bucket เสร็จแล้วจะได้หน้าตาแบบนี้

![create bucket success](./asset-5.png)

เข้าไปที่ bucket ที่เราสร้างไว้ ( reactcircleci.com ) เลือกหัวข้อ **Permissions -> Bucket Policy** ทำการเพิ่ม JSON เข้าไป ( เพื่อบอกให้ bucket นี้สามารถเข้าถึง content ผ่าน browser ได้ ) แล้ว กด Save

```
#Bucket Policy JSON

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AddPerm",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::reactcircleci.com/*"
        }
    ]
}
```

จากนั้นเข้าไปที่ **My Security Credentials**

![security credentials](./asset-6.png)

ให้ทำการสร้าง **Access keys (access key ID and secret access key) -> Create New Access Key**

![create access key success](./asset-7.png)

รันคำสั่ง aws configure แล้วทำการ เพิ่ม access id และ secret id เข้าไป

```
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE 
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY 
Default region name [None]: us-west-2 
Default output format [None]: json
```

ไปที่หัวข้อ **Properties -> Static Web Hosting** ตั้งค่าตามรูปข้างล่าง แล้ว กด Save

![config static website hosting](./asset-8.png)

ทำการติดตั้ง aws cli ผ่าน homebrew

```
$ brew install awscli
```

ถ้ายังไม่ได้ติดตั้ง homebrew ให้ติดตั้งตามลิ้งนี้ได้เลย [https://brew.sh/index\_th](https://brew.sh/index_th)

จากนั้นให้ build react project ( reactcircleci ) เพื่อทำการ sync กับ aws s3 bucket เพื่อทดสอบว่าทำ authentication สำเร็จ

```
$ yarn build
$ aws s3 sync build/ s3://myawesomeapp.com --delete

(aws s3 sync <% path build folder %> s3://<% bucket-name %>/)
```

( — delete หมายถึงให้ลบ ตัว build ที่อยู่ข้างใน bucket ออกก่อน แล้วค่อย sync ตัวใหม่ลงไป)

โดยถ้าทำการ sync สำเร็จ จะได้หน้าตา command line ประมาณนี้

![aws sync success](./asset-9.png)

ทำการทดสอบ เข้า website ผ่าน Endpoint ที่ได้จากการ Setup **Static website hosting** ก่อนหน้านี้

![Endpoint : [http://reactcircleci.com.s3-website-ap-southeast-1.amazonaws.com](http://reactcircleci.com.s3-website-ap-southeast-1.amazonaws.com)](./asset-10.png)

จะเห็นหน้า website เป็น ค่าเริ่มต้นของ create-react-app

![react app on aws s3](./asset-11.png)

ให้เพิ่มคำสั่ง **deploy** เข้าไปใน package.json ( สำหรับใช้รันใน circle ci )

```
{
  "name": "react-circleci",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "react": "^16.3.1",
    "react-dom": "^16.3.1",
    "react-scripts": "1.1.4"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --env=jsdom",
    "eject": "react-scripts eject",
    "deploy": "yarn build && aws s3 sync build/ s3://reactcircleci.com --delete"
  }
}
```

---

#### CircleCI Setup

ทำการสร้าง account แล้วทำการ login ถ้า login สำเร็จจะถูก redirect ไปที่ [https://circleci.com/dashboard](https://circleci.com/dashboard) ให้เลือกหัวข้อ **Add Project** แล้วเลือก repository ที่ต้องการ follow โดยคลิกที่ **Set Up Project**

![add project to circle ci](./asset-12.png)

หลังจากนั้นจะเข้ามาหน้า set up project เลือก Linux และ Node

![set up circle ci](./asset-13.png)

ทำการสร้าง folder .circleci และ ไฟล์ config.yml ผ่านคำสั่ง

```
$ mkdir .circleci && touch ./circleci/config.yml
```

โดยให้เขียนคำสั่งใน config.yml ดังนี้

```
# Javascript Node CircleCI 2.0 configuration file
#
# Check https://circleci.com/docs/2.0/language-javascript/ for more details
#
version: 2
jobs:
  build:
    docker:
      # specify the version you desire here
      - image: circleci/node:7.10
      
      # Specify service dependencies here if necessary
      # CircleCI maintains a library of pre-built images
      # documented at https://circleci.com/docs/2.0/circleci-images/
      # - image: circleci/mongo:3.4.4

working_directory: ~/repo

steps:
      # Checkout the code from the branch into the working_directory
      - checkout
      # Log the current branch
      - run:
          name: Show current branch
          command: echo ${CIRCLE_BRANCH}

# Download and cache dependencies
      - restore_cache:
          keys:
          - v1-dependencies-{{ checksum "package.json" }}
          # fallback to using the latest cache if no exact match is found
          - v1-dependencies-

- run: yarn install

- save_cache:
          paths:
            - node_modules
          key: v1-dependencies-{{ checksum "package.json" }}
        
      # run tests!
      - run: yarn test
      - run: sudo apt-get update && sudo apt-get install -y python-dev
      - run: sudo curl -O https://bootstrap.pypa.io/get-pip.py
      - run: sudo python get-pip.py
      - run: sudo pip install awscli --upgrade
      - run: aws --version
      - run: aws s3 ls
      - run: yarn run deploy
```

คำสั่งข้างบนจะมีหน้าที่คร่าวๆดังนี้

-   ทำการ checkout คือ โหลด source code ล่าสุดมาจาก repository
-   log brach ของ repository ออกมา
-   ถ้าเจอ cache อยู่ให้เอา cache นั้นมาใช้
-   ทำการ yarn install
-   ทำการ save cahche ในที่นี้คือ node\_modules
-   ทำการ yarn run test
-   ติดตั้ง aws cli
-   ทำการ yarn run deploy

กดปุ่ม **Start Building**

![start building](./asset-14.png)

หลังจากนั้นไปที่ **Project Settings -> AWS Permissions** ให้นำ **AWS\_ACCESS\_KEY\_ID** และ **AWS\_SECRET\_ACCESS\_KEY ที่สร้างขึ้นมาก่อนหน้านี้มาใช้ (** สามารถ ใช้คำสั่ง **vim ~/.aws/credentials** เพื่อเข้าไปดู key ได้ **)** แล้วกด **Save AWS keys**

![aws keys in circle ci](./asset-15.png)

หลังจากนั้นให้ commit file ที่ใช้ config circleci ขึ้นไปที่ github repository แล้วดูผลลัพธ์ในหน้า circleci/dashboard

![circle ci dashboard](./asset-16.png)

เมื่อเราทำการ commit ไฟล์ขึ้น repository แล้ว ตัว github จะทำการ trigger ไปหา circleci แล้ว circleci จะทำการรันคำสั่งตามที่เรากำหนดไว้ใน config.yml หากสำเร็จจะขึ้น SUCCESS หากไม่สำเร็จจะขึ้น FAILED ( เราสามารถส่ง message succes หรือ failed ไปหา third-party อื่นๆ ได้ เช่น line, slack, email)

หลังจากนี้ทุกครั้งที่มีการ commit file จะทำการรันคำสั่งใน circleci โดยเราสามารถแยกได้ว่า ให้รัน circleci เฉพาะ branch ที่เรากำหนด ตัวอย่างประมาณนี้ ( โดย ${CIRCLE\_BRANCH} จะทำหน้าที่ get ชื่อ branch ของ repository ออกมา)

```
# Build project with different configuration based on the current branch
- run:
    name: Building
    command: |
        if [ "${CIRCLE_BRANCH}" == "staging" ]; then
            yarn build-stg
        elif [ "${CIRCLE_BRANCH}" == "master" ]; then
            yarn build-prod
        else
            yarn build-dev
        fi
```

---

#### Setting AWS CloudFront

สังเกตว่า ตัว aws ที่เรา deploy ไปก่อนหน้านี้ ( [http://reactcircleci.com.s3-website-ap-southeast-1.amazonaws.com/](http://reactcircleci.com.s3-website-ap-southeast-1.amazonaws.com/) )ยังไม่เป็น https ให้เข้าไปที่ [https://console.aws.amazon.com/cloudfront/home](https://console.aws.amazon.com/cloudfront/home) เลือก **Create Distribution -> Get Started** แล้วทำการ setting ดังนี้ แล้วกด **Create Distribution**

![config aws cloudfront](./asset-17.png)

-   **Origin Domain Name :** reactcircleci.com.s3-website-ap-southeast-1.amazonaws.com (endpoint ของ s3)
-   **Origin ID:** S3-reactcircleci.com
-   **Viewer Protocol Policy:** Redirect HTTP to HTTPS

![config aws cloudfront](./asset-18.png)

-   **Default Root Object:** index.html

เมื่อเสร็จแล้วจะได้หน้าตาแบบนี้ พร้อมกับ status: In Progress รอประมาณ 15–20 นาที ถึงจะขึ้น status: Deployed

![config aws cloudfront](./asset-19.png)

หลังจากขึ้น status: Deployed สามารถเข้าดูผลลัพธ์ได้จาก Domain Name [https://dhhqbyhy7gv01.cloudfront.net/](https://dhhqbyhy7gv01.cloudfront.net/) จะพบว่าเป็น https แล้ว

![react app https with aws cloudfront](./asset-20.png)

---

#### To do next …

-   ทดสอบ Circle CI กับ platform อื่นๆ ( iOS, Android )
-   [netlify](https://www.netlify.com/) เป็นหนึ่งในตัวเลือกที่น่าสนใจหากมี repository บน Gitlab ( CircleCI ยังทำได้แค่บน Github และ Bitbucket )
-   [NOW](https://zeit.co/now) เป็นอีกตัว Deploy ที่ set up ง่ายกว่า aws รวมถึงมีการทำ https ให้เลย จากที่ลองทำ ยังมีติดอยู่ตรงที่ เมื่อเราทำการ `now —-public` จะได้ dynamic domain มาแบบนี้ [https://circleci-deployment-sfdfayelld.now.sh/](https://circleci-deployment-sfdfayelld.now.sh/) สามารถทำ alias ได้แบบนี้ [https://circleci-deployment-production.now.sh/](https://circleci-deployment-production.now.sh/) แต่ยังติดตรงให้มันทำ alias auto บน ci ไม่ได้

---

#### Conclusion

-   ในตัวของ CircleCI ยังสามารถทำอะไรได้มากกว่านี้อีกมาก
-   จะเห็นว่าการทำ CI / CD ในขั้นตอนที่เริ่มนั้นใช้เวลาพอสมควร แต่ก็แลกมาด้วยความคุ้มค่าในระยะยาว
-   เมื่อมีการเริ่มต้น project ใหม่ ควรเริ่มต้นวางโครง CI / CD ไว้เลย
-   การทำ run test, build, deploy แบบ Manual เป็น cost ที่ไม่ควรเอาเวลาทำอย่างอื่นของเราไปแลก
-   การทำ CI / CD จะสามารถหา Bug Creator ได้ง่ายมาก ฮ่าๆ
-   Alternatives CI -> Jenkins , Travis CI

สุดท้ายมีข้อผิดพลาด หรือ ควรเพิ่มเติมตรงไหน สามารถแนะนำเข้ามาได้เลยครับ :)

---

#### References

-   [CI & CD คืออะไร ?](https://medium.com/@narissara.tan/%E0%B8%81%E0%B8%B2%E0%B8%A3%E0%B8%99%E0%B8%B3-automated-testing-%E0%B8%A1%E0%B8%B2%E0%B8%A3%E0%B8%A7%E0%B8%A1%E0%B9%80%E0%B8%82%E0%B9%89%E0%B8%B2%E0%B8%81%E0%B8%B1%E0%B8%9A-%E0%B8%81%E0%B8%A3%E0%B8%B0%E0%B8%9A%E0%B8%A7%E0%B8%99%E0%B8%81%E0%B8%B2%E0%B8%A3-ci-cd-58c97e31e230)
-   [Continuous Integration. CircleCI vs Travis CI vs Jenkins](https://hackernoon.com/continuous-integration-circleci-vs-travis-ci-vs-jenkins-41a1c2bd95f5)
-   [Speed Up Your npm Workflow With npx](https://alligator.io/workflow/npx/)
-   [How we used CircleCI 2.0 to build and deploy an Angular app to AWS S3](https://medium.freecodecamp.org/our-journey-for-using-circleci-2-0-to-build-and-deploy-an-angular-app-to-aws-s3-8e7ea3f51503)
-   [React with CircleCI, AWS S3 and AWS CloudFront](https://blog.cloudboost.io/react-with-circleci-aws-s3-and-aws-cloudfront-844a1b2c75c9)
-   [Automating with CircleCi 2.0](https://medium.com/@RockChalkDev/automating-with-circleci-2-0-57de968bda90)
