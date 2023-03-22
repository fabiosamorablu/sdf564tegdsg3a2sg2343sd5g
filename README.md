{
  "schemaVersion": "0.3",
  "description": "Brings EC2 Instance into compliance with standing Baseline; rolls back root Volume on failure.",
  "assumeRole": "{{AutomationAssumeRole}}",
  "parameters": {
    "InstanceId": {
      "type": "String",
      "description": "(Required) EC2 InstanceId to which we apply the patch-baseline"
    },
    "ReportS3Bucket": {
      "type": "String",
      "description": "(Optional) S3 Bucket destination for the full Compliance Report generated during process",
      "default": ""
    },
    "AutomationAssumeRole": {
      "type": "String",
      "description": "(Optional) The ARN of the role that allows Automation to perform the actions on your behalf.",
      "default": ""
    },
    "LambdaAssumeRole": {
      "type": "String",
      "description": "(Optional) The ARN of the role that allows Lambda created by Automation to perform the actions on your behalf. If not specified a transient role will be created to execute the Lambda function.",
      "default": ""
    }
  },
  "mainSteps": [
    {
      "name": "createDocumentStack",
      "maxAttempts": 8,
      "action": "aws:createStack",
      "inputs": {
        "Capabilities": [
          "CAPABILITY_IAM"
        ],
        "ClientRequestToken": "{{automation:EXECUTION_ID}}",
        "StackName": "PatchInstanceWithRollbackStack{{automation:EXECUTION_ID}}",
        "Parameters": [
          {
            "ParameterKey": "LambdaRoleArn",
            "ParameterValue": "{{LambdaAssumeRole}}"
          },
          {
            "ParameterKey": "IdentifyRootVolumeLambdaName",
            "ParameterValue": "IdentifyRootVolumeLambda-{{automation:EXECUTION_ID}}"
          },
          {
            "ParameterKey": "SleepThruInstallationLambdaName",
            "ParameterValue": "SleepThruInstallationLambda-{{automation:EXECUTION_ID}}"
          },
          {
            "ParameterKey": "CheckComplianceLambdaName",
            "ParameterValue": "CheckComplianceLambda-{{automation:EXECUTION_ID}}"
          },
          {
            "ParameterKey": "SaveComplianceReportToS3LambdaName",
            "ParameterValue": "SaveRptToS3Lambda-{{automation:EXECUTION_ID}}"
          },
          {
            "ParameterKey": "ReportFailureLambdaName",
            "ParameterValue": "ReportFailureLambda-{{automation:EXECUTION_ID}}"
          },
          {
            "ParameterKey": "RestoreFromSnapshotLambdaName",
            "ParameterValue": "RestoreFromSnapshotLambda-{{automation:EXECUTION_ID}}"
          },
          {
            "ParameterKey": "DeleteSnapshotLambdaName",
            "ParameterValue": "DeleteSnapshotLambda-{{automation:EXECUTION_ID}}"
          }
        ],
        "TemplateBody": "AWSTemplateFormatVersion: '2010-09-09'\nConditions:\n  LambdaAssumeRoleNotSpecified:\n    Fn::Or:\n    - Fn::Equals:\n      - {Ref: LambdaRoleArn}\n      - ''\n    - Fn::Equals:\n      - {Ref: LambdaRoleArn}\n      - undefined\nParameters:\n  CheckComplianceLambdaName: {Description: 'The lambda function name for the predicate\n      which evaluates the state of the EC-2 operand''s patch state compliance\n\n      ', Type: String}\n  DeleteSnapshotLambdaName: {Description: 'The lambda function name for the closings\n      cleanup predicate (case in which the EC-2 operand is deemed Compliant)\n\n      ', Type: String}\n  IdentifyRootVolumeLambdaName: {Description: 'The lambda function name for the predicate\n      which acquires the id of the root Volume of the operand EC2 Instance.\n\n      ', Type: String}\n  LambdaRoleArn: {Default: '', Description: 'The ARN of the role that allows Lambda\n      created by Automation to perform the action on your behalf\n\n      ', Type: String}\n  ReportFailureLambdaName: {Description: 'The lambda function name for the predicate\n      which reports failure upon detection execution failure state indicators\n\n      ', Type: String}\n  RestoreFromSnapshotLambdaName: {Description: 'The lambda function name for the recovery\n      action predicate (case in which the EC-2 operand is deemed NonCompliant)\n\n      ', Type: String}\n  SaveComplianceReportToS3LambdaName: {Description: 'The lambda function name for\n      the predicate which consumes a serialized Compliance report and saves it in\n      S3\n\n      ', Type: String}\n  SleepThruInstallationLambdaName: {Description: 'The lambda function name for the\n      predicate which monitors the installation, deferring subsequent step procession\n      until the installations are completed or the Lambda exhausts its allotted lifespan\n\n      ', Type: String}\nResources:\n  CheckComplianceLambda:\n    Properties:\n      Code: {ZipFile: \"import boto3\\nfrom botocore.config import Config\\nimport json\\n\\\n          import sys\\nfrom datetime import datetime\\n\\nMAX_REPORT_ITEMS = 1200\\nSUCCESS_MESSAGE\\\n          \\ = \\\"Instance patched successfully\\\"\\nFAILURE_MESSAGE = \\\"Patching failed\\\n          \\ and Instance restored to original state\\\"\\nLIMITED_REPORT_DESCRIPTION_MESSAGE\\\n          \\ = \\\"Listing last %d applicable compliance items. To get the full report\\\n          \\ use the option to persist report in S3\\\" % MAX_REPORT_ITEMS\\nFULL_REPORT_DESCRIPTION_MESSAGE\\\n          \\ = \\\"Listing all compliance items.\\\"\\nconfig = Config(\\n\\tretries = {\\n\\\n          \\t\\t'max_attempts': 20,\\n\\t\\t'mode': 'standard'\\n\\t}\\n)\\nssm = boto3.client('ssm',\\\n          \\ config=config)\\ns3 = boto3.resource('s3')\\n\\ndef is_compliant(ec2_id):\\n\\\n          \\treport = {\\n\\t\\t\\\"is_compliant\\\": None,\\n\\t\\t\\\"items\\\": [],\\n\\t}\\n\\tnoncompliance_found\\\n          \\ = False\\n\\ttoken = None\\n\\twhile 1:\\n\\t\\tif token is None:\\n\\t\\t\\tlist_compliance_result\\\n          \\ = ssm.list_compliance_items(\\n\\t\\t\\t\\tFilters=[{'Key': 'ComplianceType',\\\n          \\ 'Values': ['Patch']}],\\n\\t\\t\\t\\tResourceIds=[ec2_id],\\n\\t\\t\\t\\tResourceTypes=[\\\"\\\n          ManagedInstance\\\"]\\n\\t\\t\\t)\\n\\n\\t\\telse:\\n\\t\\t\\tlist_compliance_result =\\\n          \\ ssm.list_compliance_items(\\n\\t\\t\\t\\tFilters=[{'Key': 'ComplianceType',\\\n          \\ 'Values': ['Patch']}],\\n\\t\\t\\t\\tResourceIds=[ec2_id],\\n\\t\\t\\t\\tResourceTypes=[\\\"\\\n          ManagedInstance\\\"],\\n\\t\\t\\t\\tNextToken=token\\n\\t\\t\\t)\\n\\t\\tcompliances =\\\n          \\ list_compliance_result[\\\"ComplianceItems\\\"]\\n\\n\\t\\tfor a_val in compliances:\\n\\\n          \\t\\t\\tpatch_item = {\\n\\t\\t\\t\\t\\\"PatchId\\\": a_val[\\\"Id\\\"],\\n\\t\\t\\t\\t\\\"Severity\\\"\\\n          : a_val[\\\"Severity\\\"],\\n\\t\\t\\t\\t\\\"ComplianceType\\\": a_val[\\\"ComplianceType\\\"\\\n          ],\\n\\t\\t\\t\\t\\\"State\\\": a_val[\\\"Details\\\"][\\\"PatchState\\\"]\\n\\t\\t\\t}\\n\\t\\t\\\n          \\treport[\\\"items\\\"].append(patch_item)\\n\\t\\t\\tif a_val[\\\"Status\\\"] != \\\"\\\n          COMPLIANT\\\" and a_val[\\\"Details\\\"][\\\"PatchState\\\"] != \\\"NotApplicable\\\"\\\n          :\\n\\t\\t\\t\\tnoncompliance_found = True\\n\\t\\tif \\\"NextToken\\\" in list_compliance_result:\\n\\\n          \\t\\t\\ttoken = list_compliance_result[\\\"NextToken\\\"]\\n\\t\\telse:\\n\\t\\t\\tbreak\\n\\\n          \\n\\t\\tif token is None:\\n\\t\\t\\tbreak\\n\\n\\treport[\\\"is_compliant\\\"] = noncompliance_found\\\n          \\ is False\\n\\treturn report\\n\\ndef is_success_case(ec2_id):\\n\\treturn is_compliant(ec2_id)\\n\\\n          \\ndef get_account_id(lambdaArn):\\n\\taccount_id = lambdaArn.split(\\\":\\\")[4]\\n\\\n          \\treturn account_id\\n\\ndef get_region(lambdaArn):\\n\\tregion = lambdaArn.split(\\\"\\\n          :\\\")[3]\\n\\treturn region\\n\\ndef save_to_s3(event, context, compliance_report):\\n\\\n          \\taccount_id = get_account_id(context.invoked_function_arn)\\n\\tregion =\\\n          \\ get_region(context.invoked_function_arn)\\n\\tdate = str(datetime.now().day)\\n\\\n          \\tmonth = str(datetime.now().month)\\n\\tyear = str(datetime.now().year)\\n\\\n          \\tfile_path = \\\"AWSLogs/\\\" + account_id + \\\"/SystemsManager/\\\" + region\\\n          \\ + \\\"/\\\" + year + \\\"/\\\" + month + \\\"/\\\" + date + \\\"/\\\" + event[\\\"ReportFileName\\\"\\\n          ]\\n\\trpt_s3_obj = s3.Object(event[\\\"S3Bucket\\\"], file_path)\\n\\trpt_s3_obj.put(Body=json.dumps(compliance_report))\\n\\\n          \\trpt_s3_obj_acl = s3.ObjectAcl(event[\\\"S3Bucket\\\"], file_path)\\n\\trpt_s3_obj_acl.put(ACL='bucket-owner-full-control')\\n\\\n          \\n\\treturn file_path\\n\\ndef filter_not_applicable_items(compliance_report):\\n\\\n          \\tapplicable_items = []\\n\\tfor item in compliance_report:\\n\\t\\tif item[\\\"\\\n          State\\\"] != \\\"NotApplicable\\\":\\n\\t\\t\\tapplicable_items.append(item)\\n\\t\\\n          return applicable_items\\n\\ndef handler(event, context):\\n\\tcompliance_report\\\n          \\ = is_success_case(event[\\\"InstanceId\\\"])\\n\\toutput = {\\n\\t\\t\\\"Result\\\"\\\n          : SUCCESS_MESSAGE if compliance_report[\\\"is_compliant\\\"] else FAILURE_MESSAGE,\\n\\\n          \\t\\t\\\"PatchingSuccess\\\": compliance_report[\\\"is_compliant\\\"]\\n\\t}\\n\\n\\t\\\n          if len(event[\\\"S3Bucket\\\"]) > 0:\\n\\t\\tfile_path = save_to_s3(event, context,\\\n          \\ compliance_report[\\\"items\\\"])\\n\\t\\toutput[\\\"ReportFileName\\\"] = file_path\\n\\\n          \\t\\toutput[\\\"S3Bucket\\\"] = event[\\\"S3Bucket\\\"]\\n\\n\\tif len(compliance_report[\\\"\\\n          items\\\"]) > MAX_REPORT_ITEMS:\\n\\t\\toutput[\\\"Description\\\"] = LIMITED_REPORT_DESCRIPTION_MESSAGE\\n\\\n          \\t\\tapplicable_items = filter_not_applicable_items(compliance_report[\\\"\\\n          items\\\"])\\n\\t\\toutput[\\\"items\\\"] = applicable_items[-MAX_REPORT_ITEMS:]\\n\\\n          \\telse:\\n\\t\\toutput[\\\"Description\\\"] = FULL_REPORT_DESCRIPTION_MESSAGE\\n\\\n          \\t\\toutput[\\\"items\\\"] = compliance_report[\\\"items\\\"]\\n\\treturn output\\n\"}\n      FunctionName: {Ref: CheckComplianceLambdaName}\n      Handler: index.handler\n      MemorySize: 128\n      Role:\n        Fn::If:\n        - LambdaAssumeRoleNotSpecified\n        - Fn::GetAtt: [LambdaRole, Arn]\n        - {Ref: LambdaRoleArn}\n      Runtime: python3.7\n      Timeout: 300\n    Type: AWS::Lambda::Function\n  DeleteSnapshotLambda:\n    Properties:\n      Code: {ZipFile: \"import boto3\\nimport time\\nfrom botocore.exceptions import\\\n          \\ ClientError\\n\\nSNAPSHOT_NOT_FOUND_ERROR_CODE = \\\"InvalidSnapshot.NotFound\\\"\\\n          \\nPOLL_INTERVAL = 5\\nPOLL_MAX_TRIES = 8\\n\\nssm = boto3.client('ssm')\\nec2\\\n          \\ = boto3.client('ec2')\\n\\n\\ndef check_for_deletion(snapshot_id):\\n\\ttry:\\n\\\n          \\t\\tec2.describe_snapshots(SnapshotIds=[snapshot_id])\\n\\texcept ClientError\\\n          \\ as api_client_error:\\n\\t\\tif api_client_error.response[\\\"Error\\\"][\\\"Code\\\"\\\n          ] == SNAPSHOT_NOT_FOUND_ERROR_CODE:\\n\\t\\t\\treturn True\\n\\t\\telse:\\n\\t\\t\\t\\\n          raise\\n\\n\\ndef wait_for_deletion(snapshot_id, max_tries=POLL_MAX_TRIES,\\\n          \\ wait_time=POLL_INTERVAL):\\n\\ttotal_iterations = 0\\n\\twhile total_iterations\\\n          \\ < max_tries:\\n\\t\\ttotal_iterations += 1\\n\\t\\tif check_for_deletion(snapshot_id)\\\n          \\ is True:\\n\\t\\t\\treturn True\\n\\t\\ttime.sleep(wait_time)\\n\\n\\ndef delete_snapshot(snapshot_id):\\n\\\n          \\tec2.delete_snapshot(SnapshotId=snapshot_id)\\n\\twait_for_deletion(snapshot_id)\\n\\\n          \\n\\ndef handler(event, context):\\n\\tsnap_id = event[\\\"PrePatchSnapshot\\\"\\\n          ]\\n\\n\\tdelete_snapshot(snap_id)\\n\\treturn {\\\"ResultCase\\\": \\\"Snapshot Deleted\\\"\\\n          }\\n\"}\n      FunctionName: {Ref: DeleteSnapshotLambdaName}\n      Handler: index.handler\n      MemorySize: 128\n      Role:\n        Fn::If:\n        - LambdaAssumeRoleNotSpecified\n        - Fn::GetAtt: [LambdaRole, Arn]\n        - {Ref: LambdaRoleArn}\n      Runtime: python3.7\n      Timeout: 60\n    Type: AWS::Lambda::Function\n  IdentifyRootVolumeLambda:\n    Properties:\n      Code: {ZipFile: \"import boto3\\n\\nec2_client = boto3.client('ec2')\\n\\ndef get_root_device_name(instance_id):\\n\\\n          \\troot_device_name = ec2_client.describe_instance_attribute(\\n\\t\\tAttribute='rootDeviceName',\\\n          \\ InstanceId=instance_id\\n\\t)[\\\"RootDeviceName\\\"][\\\"Value\\\"]\\n\\treturn root_device_name\\n\\\n          \\ndef get_root_volume_id_of_ec2(instance_id):\\n\\troot_device_name = get_root_device_name(instance_id)\\n\\\n          \\tvolume_id = ec2_client.describe_volumes(\\n\\t\\tFilters=[{'Name': 'attachment.instance-id',\\\n          \\ 'Values': [instance_id]},\\n\\t\\t\\t\\t {'Name': 'attachment.device', 'Values':\\\n          \\ [root_device_name]}]\\n\\t)[\\\"Volumes\\\"][0][\\\"VolumeId\\\"]\\n\\treturn volume_id\\n\\\n          \\n\\ndef handler(event, context):\\n\\tinstance_id = event[\\\"InstanceId\\\"]\\n\\\n          \\treturn get_root_volume_id_of_ec2(instance_id)\\n\"}\n      FunctionName: {Ref: IdentifyRootVolumeLambdaName}\n      Handler: index.handler\n      MemorySize: 128\n      Role:\n        Fn::If:\n        - LambdaAssumeRoleNotSpecified\n        - Fn::GetAtt: [LambdaRole, Arn]\n        - {Ref: LambdaRoleArn}\n      Runtime: python3.7\n      Timeout: 128\n    Type: AWS::Lambda::Function\n  LambdaRole:\n    Condition: LambdaAssumeRoleNotSpecified\n    Properties:\n      AssumeRolePolicyDocument:\n        Statement:\n        - Action: ['sts:AssumeRole']\n          Effect: Allow\n          Principal:\n            Service: [lambda.amazonaws.com]\n        Version: '2012-10-17'\n      Path: /\n      Policies:\n      - PolicyDocument:\n          Statement:\n            Action: ['ec2:CreateSnapshot', 'ec2:DeleteSnapshot', 'ec2:DescribeInstances',\n              'ec2:DescribeSnapshots', 'ec2:DescribeVolumes', 'ec2:DescribeInstanceAttribute',\n              'ec2:CreateVolume', 'ec2:AttachVolume', 'ec2:DetachVolume', 'ec2:StartInstances',\n              'ec2:StopInstances', 'ssm:ListCommands', 'ssm:ListComplianceItems',\n              's3:PutObject', 's3:PutObjectAcl', 'cloudwatch:*', 'logs:CreateLogGroup',\n              'logs:CreateLogStream', 'logs:PutLogEvents']\n            Effect: Allow\n            Resource: '*'\n          Version: '2012-10-17'\n        PolicyName: PatchInstanceWithRollbackLambdaPolicy\n    Type: AWS::IAM::Role\n  ReportFailureLambda:\n    Properties:\n      Code: {ZipFile: \"\\n\\ndef handler(event, context):\\n\\tcompliance_report = event[\\\"\\\n          CheckCompliance\\\"]\\n\\tif compliance_report[\\\"PatchingSuccess\\\"]:\\n\\t\\treturn\\\n          \\ \\\"Patching Succeeded!\\\"\\n\\telse:\\n\\t\\traise Exception(\\\"Patching Failed\\\"\\\n          )\\n\"}\n      FunctionName: {Ref: ReportFailureLambdaName}\n      Handler: index.handler\n      MemorySize: 128\n      Role:\n        Fn::If:\n        - LambdaAssumeRoleNotSpecified\n        - Fn::GetAtt: [LambdaRole, Arn]\n        - {Ref: LambdaRoleArn}\n      Runtime: python3.7\n      Timeout: 60\n    Type: AWS::Lambda::Function\n  RestoreFromSnapshotLambda:\n    Properties:\n      Code: {ZipFile: \"import boto3\\nimport time\\n\\nssm = boto3.client('ssm')\\nec2_client\\\n          \\ = boto3.client('ec2')\\nec2_rsc = boto3.resource('ec2')\\n\\n\\ndef count_vols_attached_to_host(vol):\\n\\\n          \\treturn len(ec2_client.describe_volumes(VolumeIds=[vol.id])[\\\"Volumes\\\"\\\n          ][0][\\\"Attachments\\\"])\\n\\n\\ndef wait_4_detach(vol):\\n\\twhile count_vols_attached_to_host(vol)\\\n          \\ > 0:\\n\\t\\ttime.sleep(5)\\n\\t\\tif count_vols_attached_to_host(vol) == 0:\\n\\\n          \\t\\t\\treturn True\\n\\n\\ndef wait_4_attach(vol):\\n\\twhile count_vols_attached_to_host(vol)\\\n          \\ < 1:\\n\\t\\ttime.sleep(5)\\n\\t\\tif count_vols_attached_to_host(vol) == 1:\\n\\\n          \\t\\t\\treturn True\\n\\n\\ndef get_root_device_name(instance_id):\\n\\troot_device_name\\\n          \\ = ec2_client.describe_instance_attribute(\\n\\t\\tAttribute='rootDeviceName',\\\n          \\ InstanceId=instance_id\\n\\t)[\\\"RootDeviceName\\\"][\\\"Value\\\"]\\n\\treturn root_device_name\\n\\\n          \\n\\ndef get_volume_id(instance_id, device_name):\\n\\tvolume_id = ec2_client.describe_volumes(\\n\\\n          \\t\\tFilters=[{'Name': 'attachment.instance-id', 'Values': [instance_id]},\\n\\\n          \\t\\t\\t\\t {'Name': 'attachment.device', 'Values': [device_name]}]\\n\\t)[\\\"\\\n          Volumes\\\"][0][\\\"VolumeId\\\"]\\n\\treturn volume_id\\n\\n\\ndef restore_vol_fm_snapshot(ec2_id,\\\n          \\ snap_id, dev, vol_id):\\n\\tvol = ec2_rsc.Volume(vol_id)\\n\\n\\tnew_vol_init\\\n          \\ = ec2_client.create_volume(AvailabilityZone=vol.availability_zone, SnapshotId=snap_id)\\n\\\n          \\tnew_vol_id = new_vol_init['VolumeId']\\n\\tnew_vol = ec2_rsc.Volume(new_vol_id)\\n\\\n          \\tec2_client.get_waiter('volume_available').wait(VolumeIds=[new_vol_id])\\n\\\n          \\n\\tec2_client.stop_instances(InstanceIds=[ec2_id])\\n\\tec2_client.get_waiter('instance_stopped').wait(InstanceIds=[ec2_id])\\n\\\n          \\n\\tec2_client.detach_volume(Device=dev, InstanceId=ec2_id, VolumeId=vol_id)\\n\\\n          \\twait_4_detach(vol)\\n\\tec2_client.attach_volume(Device=dev, InstanceId=ec2_id,\\\n          \\ VolumeId=new_vol_id)\\n\\twait_4_attach(new_vol)\\n\\n\\tec2_client.start_instances(InstanceIds=[ec2_id])\\n\\\n          \\tec2_client.get_waiter('instance_running').wait(InstanceIds=[ec2_id])\\n\\\n          \\n\\treturn True\\n\\n\\ndef handler(event, context):\\n\\tinstance_id = event[\\\"\\\n          InstanceId\\\"]\\n\\tsnap_id = event[\\\"SnapshotId\\\"]\\n\\troot_device_name = get_root_device_name(instance_id)\\n\\\n          \\troot_volume_id = get_volume_id(instance_id, root_device_name)\\n\\n\\trestore_vol_fm_snapshot(instance_id,\\\n          \\ snap_id, root_device_name, root_volume_id)\\n\\treturn {\\\"ResultCase\\\":\\\n          \\ \\\"Restoration Applied\\\"}\"}\n      FunctionName: {Ref: RestoreFromSnapshotLambdaName}\n      Handler: index.handler\n      MemorySize: 128\n      Role:\n        Fn::If:\n        - LambdaAssumeRoleNotSpecified\n        - Fn::GetAtt: [LambdaRole, Arn]\n        - {Ref: LambdaRoleArn}\n      Runtime: python3.7\n      Timeout: 300\n    Type: AWS::Lambda::Function\n  SaveComplianceReportToS3Lambda:\n    Properties:\n      Code: {ZipFile: \"\\ndef handler(event, context):\\n\\tif (len(event[\\\"S3Bucket\\\"\\\n          ]) > 0):\\n\\t\\treturn {\\n\\t\\t\\t\\\"ReportFileName\\\": event[\\\"CheckCompliance\\\"\\\n          ][\\\"ReportFileName\\\"],\\n\\t\\t\\t\\\"S3Bucket\\\": event[\\\"S3Bucket\\\"]\\n\\t\\t}\\n\\\n          \\telse:\\n\\t\\treturn None\\n\"}\n      FunctionName: {Ref: SaveComplianceReportToS3LambdaName}\n      Handler: index.handler\n      MemorySize: 128\n      Role:\n        Fn::If:\n        - LambdaAssumeRoleNotSpecified\n        - Fn::GetAtt: [LambdaRole, Arn]\n        - {Ref: LambdaRoleArn}\n      Runtime: python3.7\n      Timeout: 60\n    Type: AWS::Lambda::Function\n  SleepThruInstallationLambda:\n    Properties:\n      Code: {ZipFile: \"import boto3\\n\\nec2_rsc = boto3.resource('ec2')\\n\\n\\ndef wait_for_restart(ec2_inst):\\n\\\n          \\tec2_inst.wait_until_running()\\n\\n\\ndef handler(event, context):\\n\\tinstance\\\n          \\ = ec2_rsc.Instance(event[\\\"InstanceId\\\"])\\n\\treturn wait_for_restart(instance)\\n\"}\n      FunctionName: {Ref: SleepThruInstallationLambdaName}\n      Handler: index.handler\n      MemorySize: 128\n      Role:\n        Fn::If:\n        - LambdaAssumeRoleNotSpecified\n        - Fn::GetAtt: [LambdaRole, Arn]\n        - {Ref: LambdaRoleArn}\n      Runtime: python3.7\n      Timeout: 300\n    Type: AWS::Lambda::Function\n"
      }
    },
    {
      "name": "IdentifyRootVolume",
      "action": "aws:invokeLambdaFunction",
      "inputs": {
        "FunctionName": "IdentifyRootVolumeLambda-{{automation:EXECUTION_ID}}",
        "Payload": "{\"InstanceId\": \"{{InstanceId}}\"}"
      }
    },
    {
      "name": "PrePatchSnapshot",
      "action": "aws:executeAutomation",
      "inputs": {
        "DocumentName": "AWS-CreateSnapshot",
        "RuntimeParameters": {
          "VolumeId": "{{IdentifyRootVolume.Payload}}",
          "Description": "ApplyPatchBaseline restoration case contingency"
        }
      }
    },
    {
      "name": "installMissingUpdates",
      "action": "aws:runCommand",
      "maxAttempts": 1,
      "onFailure": "Continue",
      "inputs": {
        "DocumentName": "AWS-RunPatchBaseline",
        "InstanceIds": [
          "{{InstanceId}}"
        ],
        "Parameters": {
          "Operation": "Install"
        }
      }
    },
    {
      "name": "SleepThruInstallation",
      "action": "aws:invokeLambdaFunction",
      "maxAttempts": 10,
      "inputs": {
        "FunctionName": "SleepThruInstallationLambda-{{automation:EXECUTION_ID}}",
        "Payload": "{\"InstanceId\": \"{{InstanceId}}\"}"
      }
    },
    {
      "name": "CheckCompliance",
      "action": "aws:invokeLambdaFunction",
      "description": "Check compliance status of instance and output report of compliance items. Output may not contain all items, the full report will be in S3 if S3Bucket is specified.",
      "inputs": {
        "FunctionName": "CheckComplianceLambda-{{automation:EXECUTION_ID}}",
        "Payload": "{\"InstanceId\": \"{{InstanceId}}\", \"S3Bucket\": \"{{ReportS3Bucket}}\", \"ReportFileName\": \"PatchInstanceWithRollback-{{automation:EXECUTION_ID}}.json\"}"
      }
    },
    {
      "name": "SaveComplianceReportToS3",
      "action": "aws:invokeLambdaFunction",
      "inputs": {
        "FunctionName": "SaveRptToS3Lambda-{{automation:EXECUTION_ID}}",
        "Payload": "{\"S3Bucket\": \"{{ReportS3Bucket}}\", \"CheckCompliance\": {{CheckCompliance.Payload}}}"
      }
    },
    {
      "name": "ReportSuccessOrFailure",
      "action": "aws:invokeLambdaFunction",
      "onFailure": "step:RestoreFromSnapshot",
      "maxAttempts": 1,
      "inputs": {
        "FunctionName": "ReportFailureLambda-{{automation:EXECUTION_ID}}",
        "Payload": "{\"CheckCompliance\": {{CheckCompliance.Payload}}}"
      },
      "nextStep": "DeleteSnapshot"
    },
    {
      "name": "RestoreFromSnapshot",
      "action": "aws:invokeLambdaFunction",
      "onFailure": "step:DeleteSnapshot",
      "inputs": {
        "FunctionName": "RestoreFromSnapshotLambda-{{automation:EXECUTION_ID}}",
        "Payload": "{\"InstanceId\": \"{{InstanceId}}\", \"SnapshotId\": \"{{PrePatchSnapshot.Output}}\", \"CheckCompliance\": {{CheckCompliance.Payload}}}"
      },
      "nextStep": "DeleteSnapshot"
    },
    {
      "name": "DeleteSnapshot",
      "action": "aws:invokeLambdaFunction",
      "onFailure": "step:deleteCloudFormationTemplate",
      "maxAttempts": 16,
      "inputs": {
        "FunctionName": "DeleteSnapshotLambda-{{automation:EXECUTION_ID}}",
        "Payload": "{\"PrePatchSnapshot\": \"{{PrePatchSnapshot.Output}}\"}"
      },
      "nextStep": "deleteCloudFormationTemplate"
    },
    {
      "name": "deleteCloudFormationTemplate",
      "action": "aws:deleteStack",
      "maxAttempts": 16,
      "inputs": {
        "StackName": "PatchInstanceWithRollbackStack{{automation:EXECUTION_ID}}"
      },
      "isEnd": "true"
    }
  ],
  "outputs": [
    "IdentifyRootVolume.Payload",
    "PrePatchSnapshot.Output",
    "SaveComplianceReportToS3.Payload",
    "RestoreFromSnapshot.Payload",
    "CheckCompliance.Payload"
  ]
}
