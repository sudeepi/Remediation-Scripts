description: '# Quarantine EC2 instances with public ports without a valid exception'
schemaVersion: '0.3'
assumeRole: '{{AutomationAssumeRole}}'
parameters:
  AutomationAssumeRole:
    type: String
  ResourceId:
    type: String
  ConfigRuleName:
    type: String
mainSteps:
  - name: quarantine_security_group
    action: 'aws:executeScript'
    inputs:
      Runtime: python3.6
      Handler: script_handler
      Script: |-
        import boto3
                ec2_client = boto3.client('ec2')
        config_client = boto3.client('config')
        def fetch_all_items(method, response_key, next_token_key, **kwargs):
            """
            client: The boto3 client
            method: The boto3 method to be fetched (e.g. ecs_client.list_clusters)
            response_key: The key name in the boto3 response that contains the relevant data without metadata (e.g. 'clusterArns')
            next_token_key: The next token key name in the boto3 response if truncated (e.g. 'nextToken')
            kwargs: The parameter name and value for a given boto3 method (e.g. ecs_client.list_services(cluster=clusterArn))
            """
            if kwargs:
                response = method(**kwargs)
            else:
                response = method()
            items = response[response_key]
            next_token = response.get(next_token_key)
            while next_token:
                token = {next_token_key: next_token}
                response = method(**token, **kwargs)
                items = items + response[response_key]
                items = items + response[response_key]
            return items
        def script_handler(events, context):
          offending_ports = []
          instance_id = events['resourceId']
          config_rule = events['configRuleName']
          evaluations = fetch_all_items(config_client.get_compliance_details_by_resource, 'EvaluationResults', 'NextToken', ResourceType='AWS::EC2::Instance', ResourceId=instance_id, ComplianceTypes=['NON_COMPLIANT'])
          for evaluation in evaluations:
              if evaluation['EvaluationResultIdentifier']['EvaluationResultQualifier']['ConfigRuleName'] == config_rule:
                  annotation = evaluation['Annotation']
                  public_ports = annotation.split('ports ')[1].split(' without')[0].strip('[').strip(']').split(', ')
                  for port in public_ports:
                      try:
                          port = int(port)
                            offending_ports.append(port)
                      except ValueError:
                          continue
          ec2_instance = ec2_client.describe_instances(InstanceIds=[instance_id])['Reservations'][0]['Instances'][0]
          all_security_groups = []
          offending_security_groups = []
          for interface in ec2_instance['NetworkInterfaces']:
              interface_id = interface['NetworkInterfaceId']
              for sg in interface['Groups']:
                  all_security_groups.append({'sg': sg['GroupId'], 'interface_id': interface_id})
          security_groups = fetch_all_items(ec2_client.describe_security_groups, 'SecurityGroups', 'NextToken', GroupIds=[sg['sg'] for sg in all_security_groups])
          for security_group in security_groups:
              is_offending_security_group = False
              permissions_without_exception = []
              for permission in security_group['IpPermissions']:
                  from_port = permission.get('FromPort')
                  to_port = permission.get('ToPort')
                  for ip in permission['IpRanges']:
                        if ip['CidrIp'] != '0.0.0.0/0':
                            continue
                  if from_port in offending_ports:
                      permissions_without_exception.append(permission)
                  # If FromPort is missing, it indicates all ports allowed
                  elif not from_port:
                      permissions_without_exception.append(permission)
                  # If FromPort and ToPort are different, it indicates a port range is allowed
                  elif from_port != to_port:
                      for port in offending_ports:
                          # Check if the offending port falls within the port range
                          if from_port <= port <= to_port:
                              permissions_without_exception.append(permission)
                  if len(permissions_without_exception) == 0:
                      continue
                  is_offending_security_group = True
              if is_offending_security_group:
                  # Add egress permissions if IpProtocol is not -1 as that is automatically populated on new SGs
                  egress_permissions = [sg for sg in security_group['IpPermissionsEgress'] if sg['IpProtocol'] != '-1']
                  sg = {'sg': security_group['GroupId'], 'egress_permissions': egress_permissions}
                  offending_security_groups.append(sg)
          offending_sg_ids = [sg['sg'] for sg in offending_security_groups]
          all_sg_ids = [sg['sg'] for sg in all_security_groups]
          if len(all_security_groups) > len(offending_security_groups):
              groups_to_maintain = list(set(all_sg_ids) - set(offending_sg_ids))
              interface_id = ''
              for sec_group in all_security_groups:
                  if sec_group['sg'] == groups_to_maintain[0]:
                      interface_id = sec_group['interface_id']
              ec2_client.modify_network_interface_attribute(NetworkInterfaceId=interface_id, Groups=groups_to_maintain)
          else:
                groups_to_maintain = list(set(all_sg_ids) - set(offending_sg_ids))
              for security_group in offending_security_groups:
                  quarantined_sg_permissions = []
                  sg = ec2_client.describe_security_groups(GroupIds=[security_group['sg']])['SecurityGroups'][0]
                  for permission in sg['IpPermissions']:
                      if permission not in permissions_without_exception:
                          quarantined_sg_permissions.append(permission)
                  try:
                    quarantined_sg = ec2_client.create_security_group(
                        GroupName=f'QUARANTINED-{sg["GroupName"]}',
                        Description=f'Quarantined security group ({sg["GroupName"]}) due to public ports without exception',
                        VpcId=sg['VpcId']
                    )
                  except:
                    quarantined_sg = {}
                      security_groups_in_vpc = ec2_client.describe_security_groups(Filters=[{'Name':'vpc-id', 'Values': [sg['VpcId']]}])
                    for sg_in_vpc in security_groups_in_vpc:
                      if sg_in_vpc['GroupName'] == f'QUARANTINED-{sg["GroupName"]}':
                        quarantined_sg = sg_in_vpc
                    if not quarantined_sg:
                      return "Could not create or find quarantined SG"
                  interface_id = ''
                  for sec_group in all_security_groups:
                      if sec_group['sg'] == security_group['sg']:
                          interface_id = sec_group['interface_id']
                  if len(quarantined_sg_permissions) > 0:
                    try:
                      ec2_client.authorize_security_group_ingress(GroupId=quarantined_sg['GroupId'], IpPermissions=quarantined_sg_permissions)
                    except:
                      pass
                  if len(security_group['egress_permissions']) > 0:
                    try:
                      ec2_client.authorize_security_group_egress(GroupId=quarantined_sg['GroupId'], IpPermissions=security_group['egress_permissions'])
                    except:
                      pass
                  groups_to_maintain.append(quarantined_sg['GroupId'])
                  ec2_client.modify_network_interface_attribute(NetworkInterfaceId=interface_id, Groups=groups_to_maintain)
      InputPayload:
          resourceId: '{{ResourceId}}'
        configRuleName: '{{ConfigRuleName}}'

    



