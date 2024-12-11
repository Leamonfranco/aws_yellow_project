## Índice
+ [Descripción del Proyecto](#descripción-del-proyecto)
+ [Servicios utilizados](#servicios-utilizados)
+ [Arquitectura](#arquitectura)
+ [Configuraciones](#configuraciones)
+ [Integrantes del Equipo](#integrantes-del-equipo)

## Descripción del Proyecto
Realización del despliegue de una aplicación web con alta disponibilidad en AWS. El objetivo principal es garantizar que la web del cliente permanezca accesible incluso si una zona de disponibilidad de AWS falla.
## Servicios utilizados
- AWS CloudFormation
- Amazon VPC (Virtual Private Cloud)
  	- Internet Gateway (IGW)
	- Route Table (Tabla de Rutas)
	- Subnet (Subredes)
- Amazon EC2 (Elastic Compute Cloud)
	- Security Groups 
	- Elastic Load Balancing (ALB - Application Load Balancer)
	- Auto Scaling
- Amazon CloudWatch
- Amazon SNS (Simple Notification Service)

## Arquitectura
<img src="web/images/diagram.jpg" alt="Diagrama de la Arquitectura del Proyecto">

## Configuraciones

### 1. Configuración de la VPC y de Internet Getawey
Se configura una VPC (Virtual Private Cloud) que define una red virtual para el proyecto. Se especifica el rango de direcciones IP (`10.0.0.0/16`) y se habilita soporte DNS para resolver nombres en la red.
``` 
LabVPC: 
	Type: AWS::EC2::VPC 
	Properties: 
		CidrBlock: !Ref LabVpcCidr 
		EnableDnsSupport: true 
		EnableDnsHostnames: true 
		Tags: 
			- Key: Name 
			  Value: Lab VPC
```

### 2. Configuración del Internet Gateway y su Conexión a la VPC
Se crea un Internet Gateway (IGW) que sirve como puerta de enlace para conectar la VPC a Internet. 
``` 
IGW: 
	Type: AWS::EC2::InternetGateway 
	Properties: 
		Tags: 
			- Key: Name 
			  Value: Lab IGW
```

Posteriormente, se asocia este IGW a la VPC utilizando un `VPCGatewayAttachment`.
``` 
VPCtoIGWConnection: 
	Type: AWS::EC2::VPCGatewayAttachment 
	DependsOn: 
		- IGW 
		- LabVPC 
	Properties: 
		InternetGatewayId: !Ref IGW 
		VpcId: !Ref LabVPC
```

### 3. Creación de la Tabla de Rutas Pública
Se define una tabla de rutas para la VPC que permitirá enrutar el tráfico saliente hacia Internet. Esta tabla es fundamental para el acceso de las subredes públicas al exterior.
```
PublicRouteTable: 
	Type: AWS::EC2::RouteTable 
	DependsOn: LabVPC 
	Properties: 
		VpcId: !Ref LabVPC 
		Tags: 
			- Key: Name 
			  Value: Public Route Table
```

### 4. Configuración de la Ruta
Se configura una ruta en la tabla de rutas pública, asignando todo el tráfico (`0.0.0.0/0`) al Internet Gateway. Esto asegura que las instancias en la VPC puedan comunicarse con Internet.
``` 
PublicRoute: 
	Type: AWS::EC2::Route 
	DependsOn: 
		- PublicRouteTable 
		- IGW 
	Properties: 
		DestinationCidrBlock: 0.0.0.0/0 
		GatewayId: !Ref IGW 
		RouteTableId: !Ref PublicRouteTable
```

### 5. Subredes
Se crean dos subredes públicas, distribuidas en diferentes zonas de disponibilidad para garantizar alta disponibilidad y redundancia. Ambas subredes están configuradas para asignar automáticamente direcciones IP públicas a las instancias creadas en ellas.
``` 
PublicSubnet1: 
	Type: AWS::EC2::Subnet 
	DependsOn: LabVPC 
	Properties: 
		VpcId: !Ref LabVPC 
		MapPublicIpOnLaunch: true 
		CidrBlock: !Ref PublicSubnetCidr1 
		AvailabilityZone: !Ref AvailabilityZone1 
		Tags: 
			- Key: Name 
			  Value: Public Subnet1

PublicSubnet2: 
	Type: AWS::EC2::Subnet 
	DependsOn: LabVPC 
	Properties: 
		VpcId: !Ref LabVPC 
		MapPublicIpOnLaunch: true 
		CidrBlock: !Ref PublicSubnetCidr2 
		AvailabilityZone: !Ref AvailabilityZone2 
		Tags: 
			- Key: Name 
			  Value: Public Subnet2
```

Además, las subredes se asocian a la tabla de rutas pública para que puedan utilizar el Internet Gateway.
```
PublicRouteTableAssociation1: 
	Type: AWS::EC2::SubnetRouteTableAssociation 
	DependsOn: 
		- PublicRouteTable 
		- PublicSubnet1 
	Properties: 
		RouteTableId: !Ref 
		PublicRouteTable SubnetId: !Ref PublicSubnet1 

PublicRouteTableAssociation2: 
	Type: AWS::EC2::SubnetRouteTableAssociation 
	DependsOn: 
		- PublicRouteTable 
		- PublicSubnet2 
	Properties: 
		RouteTableId: !Ref PublicRouteTable 
		SubnetId: !Ref PublicSubnet2
```

### 6. Configuración de los grupos de seguridad
Se configuran grupos de seguridad para controlar el tráfico de red hacia y desde los recursos.

**Grupo de Seguridad del Load Balancer:** Permite tráfico HTTP (puerto 80) desde cualquier origen.
```
ELBSecurityGroup: 
	Type: AWS::EC2::SecurityGroup 
	Properties: 
		GroupDescription: ELB Security Group 
		VpcId: !Ref LabVPC 
		SecurityGroupIngress: 
			- IpProtocol: tcp 
			  FromPort: 80 
			  ToPort: 80 
			  CidrIp: 0.0.0.0/0
```

**Grupo de Seguridad de las Instancias EC2:** Permite tráfico SSH (puerto 22) para administración y tráfico HTTP (puerto 80) desde el Load Balancer.
```
EC2SecurityGroup: 
	Type: AWS::EC2::SecurityGroup 
	Properties: 
		GroupDescription: EC2 Security Group
		VpcId: !Ref LabVPC 
		SecurityGroupIngress: 
			- IpProtocol: tcp 
			  FromPort: 22 
			  ToPort: 22 
			  CidrIp: 0.0.0.0/0 
			- IpProtocol: tcp 
			  FromPort: 80 
			  ToPort: 80 
			  SourceSecurityGroupId: 
				Fn::GetAtt: 
				- ELBSecurityGroup 
				- GroupId
```

### 7. Configuración del Application Load Balancer
Se configura un Application Load Balancer que distribuye el tráfico entrante entre las instancias EC2.

**Target group**: Define el conjunto de instancias EC2 que procesarán el tráfico distribuido por el Load Balancer.
- Configura **Health Checks** en el puerto `80` con protocolo `HTTP` para verificar que las instancias estén disponibles.
- Establece un matcher que espera un código HTTP `200` como respuesta exitosa.
- Especifica la **VPC** donde se encuentran las instancias.
```
EC2TargetGroup: 
	Type: AWS::ElasticLoadBalancingV2::TargetGroup 
	Properties: 
		HealthCheckIntervalSeconds: 30 
		HealthCheckProtocol: HTTP 
		HealthCheckTimeoutSeconds: 15 
		HealthyThresholdCount: 5 
		Matcher: 
			HttpCode: '200' 
		Name: EC2TargetGroup 
		Port: 80 
		Protocol: HTTP 
		VpcId: !Ref LabVPC
```

**Application Load Balancer** es el balanceador que recibe tráfico desde Internet y lo distribuye al Target Group.
- Configurado como `internet-facing`, permite acceso externo.
- Implementado en las dos subredes públicas, lo que asegura **redundancia y alta disponibilidad**.
- Asociado a un Security Group que habilita tráfico HTTP (puerto `80`) desde cualquier origen.
```
ApplicationLoadBalancer: 
	Type: AWS::ElasticLoadBalancingV2::LoadBalancer 
	Properties: 
		Scheme: internet-facing 
		Subnets: 
			- !Ref PublicSubnet1 
			- !Ref PublicSubnet2 
		SecurityGroups: 
			- !GetAtt ELBSecurityGroup.GroupId
```

**Agente escucha o listener** es la interfaz del ALB que escucha el tráfico entrante y lo redirige.
- Configurado para escuchar en el **puerto `80`** con protocolo `HTTP`.
- Redirige el tráfico al **Target Group**, distribuyéndolo entre las instancias EC2 saludables según las reglas de balanceo.
```
ALBListener: 
	Type: AWS::ElasticLoadBalancingV2::Listener 
	Properties: 
		DefaultActions: 
			- Type: forward 
			  TargetGroupArn: !Ref EC2TargetGroup 
		LoadBalancerArn: !Ref ApplicationLoadBalancer 
		Port: 80 
		Protocol: HTTP
```

### 8. Grupo AutoScaling, Cloud Watch y Notificación por correo
Se implementa una solución de escalado automático para gestionar la cantidad de instancias EC2 según la carga del sistema:

**Plantilla de lanzamiento:** Especifica los detalles de las instancias, como tipo, configuración de red y datos de usuario para inicialización.
```
LaunchTemplate: 
	Type: AWS::EC2::LaunchTemplate 
	Properties: 
		LaunchTemplateName: !Sub ${AWS::StackName}-launch-template 
		LaunchTemplateData: 
			ImageId: !Ref LatestAmiId 
			InstanceType: !Ref InstanceType 
			KeyName: !Ref KeyName 
			SecurityGroupIds: 
				- !Ref EC2SecurityGroup 
			UserData: 
				Fn::Base64: !Sub | 
					#!/bin/bash 
					yum update -y 
					sudo amazon-linux-extras install epel -yinternal: -y 
					yum install -y httpd git stress 
					systemctl start httpd 
					systemctl enable httpd 
					git clone https://github.com/mariandrean/aws_yellow_project.git
					cd aws_yellow_project/web
					echo "const ip='$(hostname -f)'" > /var/www/html/script.js
					sudo mv index.html styles.css /var/www/html/
```

**Grupo de Auto Scaling:** Define los límites mínimo y máximo de instancias.
```
WebServerGroup: 
	Type: AWS::AutoScaling::AutoScalingGroup 
	Properties: 
		LaunchTemplate: 
			LaunchTemplateId: !Ref 
			LaunchTemplate Version: !GetAtt LaunchTemplate.LatestVersionNumber 
		MaxSize: '4' 
		MinSize: '2' 
		TargetGroupARNs: 
			- !Ref EC2TargetGroup 
		VPCZoneIdentifier: 
			- !Ref PublicSubnet1 
			- !Ref PublicSubnet2
```

**Políticas de escalado:**  Reglas configuradas para realizar el autoescalado de manera automática.
- Escalado hacia arriba (aumentar instancias) cuando la CPU supera el 70%.
- Escalado hacia abajo (reducir instancias) cuando la CPU baja del 40%.
- **CloudWatch:** Configura alarmas que supervisan el rendimiento y ejecutan las políticas de escalado.

Politica de aumento y notificación mediante CloudWatch.
```
ScalingOutPolicy: 
	Type: 'AWS::AutoScaling::ScalingPolicy' 
	Properties: 
		AdjustmentType: ChangeInCapacity 
		AutoScalingGroupName: !Ref WebServerGroup 
		Cooldown: '60' 
		ScalingAdjustment: '1' 
		
CloudWatchOutAlarm: 
	Type: 'AWS::CloudWatch::Alarm' 
	Properties: 
		EvaluationPeriods: '1' 
		Statistic: Average 
		Threshold: '70' 
		AlarmDescription: Alarm set fo 70% of CPU utilization 
		Period: '60' 
		AlarmActions: 
			- !Ref ScalingOutPolicy 
			- !Ref NotificationTopic 
		Namespace: AWS/EC2 
		Dimensions: 
			- Name: AutoScalingGroupName 
			  Value: !Ref WebServerGroup 
		ComparisonOperator: GreaterThanThreshold 
		MetricName: CPUUtilization
```

Pólitica de reducción y notificación mediante CloudWatch.
```
ScalingInPolicy: 
	Type: 'AWS::AutoScaling::ScalingPolicy' 
	Properties: 
		AdjustmentType: ChangeInCapacity 
		AutoScalingGroupName: !Ref WebServerGroup 
		Cooldown: '60' 
		ScalingAdjustment: '-1' 

CloudWatchInAlarm: 
	Type: 'AWS::CloudWatch::Alarm' 
	Properties: 
		EvaluationPeriods: '1' 
		Statistic: Average 
		Threshold: '40' 
		AlarmDescription: Alarm set for CPU utilization <= 40% 
		Period: '60' 
		AlarmActions: 
			- !Ref ScalingInPolicy 
			- !Ref NotificationTopic 
		Namespace: AWS/EC2 
		Dimensions: 
			- Name: AutoScalingGroupName 
			  Value: !Ref WebServerGroup 
		ComparisonOperator: LessThanOrEqualToThreshold 
		MetricName: CPUUtilization
```

**SNS (Simple Notification Service):** Envía notificaciones por correo sobre cambios en el estado del Auto Scaling.
```
NotificationTopic: 
	Type: AWS::SNS::Topic 
	Properties: 
		DisplayName: Auto Scaling Notifications 
		Subscription: 
			- Protocol: email 
			  Endpoint: !Ref OperatorEmail
```

### 9. Configuración del Punto de Acceso mediante el Load Balancer
Se expone el DNS del Load Balancer en las salidas (outputs) de la configuración. Este punto de acceso permite el tráfico entrante desde Internet hacia las instancias a través del Load Balancer.
```
Outputs: 
	LoadBalancerDNS: 
		Description: DNS Name of the load balancer 
		Value: !GetAtt ApplicationLoadBalancer.DNSName
```

<!-- TEAM -->
## Integrantes del Equipo

<table align='center'>
  <tr>
  <td align='center'>
      <div >
        <h4 style="margin-top: 1rem;">Leandra Montoya Franco</h4>
        <div style='display: flex; flex-direction: column'>
        <a href="https://github.com/Leamonfranco" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white"/>
        </a>
        <a href="https://www.linkedin.com/in/leandramontoya/" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/linkedin%20-%230077B5.svg?&style=for-the-badge&logo=linkedin&logoColor=white"/>
        </a>
        </div>
      </div>
    </td>
    <td align='center'>
      <div >
        <h4 style="margin-top: 1rem;">Triana Soler Martín</h4>
        <div style='display: flex; flex-direction: column'>
        <a href="https://github.com/TrianaSolerMartin" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white"/>
        </a>
        <a href="https://www.linkedin.com/in/triana-soler-martín" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/linkedin%20-%230077B5.svg?&style=for-the-badge&logo=linkedin&logoColor=white"/>
        </a>
        </div>
      </div>
    </td>
    <td align='center'>
      <div >
        <h4 style="margin-top: 1rem;">Scarlet Gonzalez García</h4>
        <div style='display: flex; flex-direction: column'>
        <a href="https://github.com/Scarlat2902/Scarlat2902" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white"/>
        </a>
        <a href="https://www.linkedin.com/in/scarlet-gonzalez-garcia/" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/linkedin%20-%230077B5.svg?&style=for-the-badge&logo=linkedin&logoColor=white"/>
        </a>
        </div>
      </div>
    </td>
    <td align='center'>
      <div >
        <h4 style="margin-top: 1rem;">Sandra Esteban López</h4>
        <div style='display: flex; flex-direction: column'>
        <a href="https://github.com/sandraEstlo" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white"/>
        </a>
        <a href="https://www.linkedin.com/in/sandra-esteban-lopez/" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/linkedin%20-%230077B5.svg?&style=for-the-badge&logo=linkedin&logoColor=white"/>
        </a>
        </div>
      </div>
    </td>
    <td align='center'>
      <div >
        <h4 style="margin-top: 1rem;">Maria Andrea An</h4>
        <div style='display: flex; flex-direction: column'>
        <a href="https://github.com/mariandrean" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white"/>
        </a>
        <a href="https://www.linkedin.com/in/mariandrean/" target="_blank">
          <img style='width:7rem' src="https://img.shields.io/badge/linkedin%20-%230077B5.svg?&style=for-the-badge&logo=linkedin&logoColor=white"/>
        </a>
        </div>
      </div>
    </td>
  </tr>
</table>
</details>
<!-- END OF TEAM -->
