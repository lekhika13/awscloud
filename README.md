#LAUNCHING AWS INFRASTRUCTURE USING TERRAFORM

1. Create the key and security group which allow the port 80.

2. Launch EC2 instance.

3. In this Ec2 instance use the key and security group which we have created in step 1.

    ---------------------------------selecting our region for instance---------------

        provider "aws" {

        region = "ap-south-1"

        profile = "lekhika"

        }

#-----------------------------------creating key----------------------------------

    resource "aws_key_pair" "taskkey1" {

    key_name = "taskkey1"

    }

#----------------------------------creating security group------------------------

    resource "aws_security_group" "tasksg" {

    name = "tasksg"

    description = "Allow TLS inbound traffic"

    vpc_id = "vpc-2a4b5742"

    ingress {

      }

      ingress {

      egress {

          }

      }


4. Launch one Volume (EBS) and mount that volume into /var/www/html

      #------------------------------------------launching ebs volume-------------------

        resource "aws_ebs_volume" "taskebs" {

        availability_zone = "ap-south-1a"

        size = 1

        tags = {

        Name = "taskebs"

        }

        }

        resource "aws_volume_attachment" "taskattach" {

        device_name = "/dev/sdf"

        volume_id = "${aws_ebs_volume.taskebs.id}"

        instance_id = "${aws_instance.lekinst.id}"

        }


5. Developer have uploded the code into github repo also the repo has some images.

6. Copy the github repo code into /var/www/html

7. Create S3 bucket, and copy/deploy the images from github repo into the s3 bucket and change the permission to public readable.

      #--------------------------------creating s3-------------------------------------

        resource "aws_s3_bucket" "b" {

        bucket = "lekhhikabalti"

       acl  = "private"

       tags = {

        Name = "My bucket"

       }

          }

          locals {

           s3_origin_id = "myS3Origin"

          }
           resource "aws_cloudfront_origin_access_identity" "origin_access_identity" {

          comment = "Some comment"

        }

        resource "aws_instance" "lekinst" {

        ami = "ami-0447a12f28fddb066"

        instance_type = "t2.micro"

        availability_zone = "ap-south-1a"

        key_name = "taskkey1"

        security_groups = [ "tasksg" ]

        user_data = <<-EOF




        #! /bin/bash

        sudo yum install httpd -y

        sudo systemctl start httpd

        sudo systemctl enable httpd

        sudo yum install git -y

        mkfs.ext4 /dev/xvdf1

        mount /dev/xvdf1 /var/www/html

        cd /var/www/html

        git clone https://github.com/lekhika13/awscloud.git

        EOF

        tags = {

        Name = "lekinst"

        }

        }
        

8 Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to update in code in /var/www/html

 #----------------------------------creating code pipeline-----------------------
        
        resource "aws_iam_role" "codepipeline_role" {
          name = "test-role"
          assume_role_policy = <<EOF
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Principal": {
                "Service": "codepipeline.amazonaws.com"
              },
              "Action": "sts:AssumeRole"
            }
          ]
        }
        EOF
        }
        resource "aws_iam_role_policy" "codepipeline_policy" {
          name = "codepipeline_policy"
          role = "${aws_iam_role.codepipeline_role.id}"
        policy = <<EOF
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect":"Allow",
              "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:GetBucketVersioning",
                "s3:PutObject"
              ],
              "Resource": [
                "${aws_s3_bucket.b.arn}",
                "${aws_s3_bucket.b.arn}/*"
              ]
            },
            {
              "Effect": "Allow",
              "Action": [
                "codebuild:BatchGetBuilds",
                "codebuild:StartBuild"
              ],
              "Resource": "*"
            }
          ]
        }
        EOF
        }
        resource "aws_codepipeline" "codepipeline" {
          name     = "tf-test-pipeline"
          role_arn = "${aws_iam_role.codepipeline_role.arn}"
         artifact_store {
          location = "${aws_s3_bucket.b.bucket}"
          type     = "S3"
        }

         stage {
          name = "Source"


          action {
            name             = "Source"
            category         = "Source"
            owner            = "ThirdParty"
            provider         = "GitHub"
            version          = "1"
            output_artifacts = ["SourceArtifact"]
            configuration = {
              Owner  = "lekhika13"
              Repo   = "awscloud"
              Branch = "master"
          OAuthToken = "bfd69100b7ebfc786b6f0e9256bbfe605193dbd8"

            }
          }
        }

        stage {
            name = "Deploy"


            action {
              name            = "Deploy"
              category        = "Deploy"
              owner           = "AWS"
              provider        = "S3"
              input_artifacts = ["SourceArtifact"]
              version         = "1"


              configuration = {
                BucketName = "${aws_s3_bucket.b.bucket}"
                Extract = "true"
              }
            }
          }
        }

#----------------------------------creating web distribution---------------------

      resource "aws_cloudfront_distribution" "s3_distribution" {
            origin {
            domain_name = "${aws_s3_bucket.b.bucket_regional_domain_name}"
            origin_id   = "${local.s3_origin_id}"
			s3_origin_config {
			  origin_access_identity = "${aws_cloudfront_origin_access_identity.origin_access_identity.cloudfront_access_identity_path}"
				}
			}
		  enabled             = true
		  is_ipv6_enabled     = true
		  comment             = "default_cache_behavior {
				allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
				cached_methods   = ["GET", "HEAD"]
				target_origin_id = "${local.s3_origin_id}"

				forwarded_values {
				  query_string = false

				  cookies {
					forward = "none"
				  }
				}

				viewer_protocol_policy = "allow-all"
				
		  }
			restrictions {
			geo_restriction {
			  restriction_type = "whitelist"
			  locations        = ["US", "CA", "GB", "IN"]
			}
			}


		  tags = {
			Environment = "production"
			}


		  viewer_certificate {
			cloudfront_default_certificate = true
		  }
    }
