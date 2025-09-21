# AWS-CLI Primer Workshop — README Internacional (ES/EN)

---

## 🧭 Resumen Ejecutivo / Executive Summary

**ES:** En este taller aprenderás a crear, conectar, listar y limpiar una instancia EC2 usando **AWS CLI** de forma reproducible y segura.  
**EN:** In this workshop you will learn to create, connect to, list, and clean up an EC2 instance using **AWS CLI** in a reproducible and secure way.

---

## 🖼️ Diagrama / Diagram

<img width="631" height="142" alt="Screenshot 2025-09-17 093229" src="https://github.com/user-attachments/assets/678b3ccc-9f86-46d9-97f8-51135524733c" /> *(Arquitectura simple: Cliente con AWS CLI → API de AWS → EC2 en una VPC con SG y Key Pair / Simple architecture: Client with AWS CLI → AWS API → EC2 in a VPC with SG and Key Pair)*

---

## ✅ Prerrequisitos / Prerequisites

- **AWS CLI** instalado / installed  
  - Descargar / Download: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- **Credenciales de AWS** con permisos mínimos (EC2 + Describe* + VPC) / AWS credentials with least-privilege (EC2 + Describe* + VPC)
- **Terminal**:
  - **Windows**: PowerShell o Git Bash
  - **macOS/Linux**: Terminal

### Configuración de AWS CLI / Configure AWS CLI

```bash
aws configure
# Example:
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: us-west-2
# Default output format [None]: json
```

> **Nota/Note:** Usa una región cercana (p. ej., `us-east-1`, `us-west-2`). Mantén tus llaves secretas **fuera** del control de versiones.


<img width="942" height="529" alt="Screenshot 2025-09-17 092141" src="https://github.com/user-attachments/assets/12e76378-9cf1-4166-a62d-71a5f78f96ee" /> *(Captura del `aws configure` exitoso / Successful `aws configure`)*

---

## 🧱 Paso 1 / Step 1 — Crear el Par de Claves EC2 / Create the EC2 Key Pair

**ES:** Este par se usa para autenticarte por SSH.  
**EN:** This key pair is used to authenticate over SSH.

```bash
aws ec2 create-key-pair   --key-name MyKeyPair   --query 'KeyMaterial'   --output text > MyKeyPair.pem

ls -la MyKeyPair.pem
```

**Salida esperada / Expected output:**
```
-rw------- 1 user staff 1675 Oct  9 20:39 MyKeyPair.pem
```

**Asegurar permisos / Secure permissions**

- **macOS/Linux:**
  ```bash
  chmod 400 MyKeyPair.pem
  ls -la MyKeyPair.pem
  ```
- **Windows (PowerShell) — alternativa a chmod / chmod alternative**:
  ```powershell
  icacls .\MyKeyPair.pem /inheritance:r
  icacls .\MyKeyPair.pem /grant:r "$($env:UserName):(R)"
  ```

**Verificar huella / Check fingerprint**
```bash
aws ec2 describe-key-pairs --key-name MyKeyPair
```


<img width="940" height="132" alt="Screenshot 2025-09-17 092536" src="https://github.com/user-attachments/assets/96e82a0b-c576-4ac7-a0b6-42cccd0e9426" /> *(Salida de `describe-key-pairs` / `describe-key-pairs` output)*

---

## 🔒 Paso 2 / Step 2 — Grupo de Seguridad / Security Group (SG)

**ES:** El SG controla qué puertos se permiten.  
**EN:** The SG controls which ports are allowed.

> **Tip:** Primero identifica la **VPC** donde crearás el SG. Si tienes **VPC predeterminada** se usa por defecto, pero aquí lo hacemos explícito.

<img width="930" height="299" alt="Screenshot 2025-09-17 092552" src="https://github.com/user-attachments/assets/ba4ebb4f-136a-46db-8724-45e114b3a0e2" /> *(Captura de reglas de SG / SG rules screenshot)*

```bash
# Listar VPCs / List VPCs
aws ec2 describe-vpcs --query 'Vpcs[].{VpcId:VpcId,Cidr:CidrBlock}' --output table
```

**Crear SG / Create SG** (reemplaza `vpc-xxxxxxxx` por tu VPC)
```bash
aws ec2 create-security-group   --group-name my-sg-cli   --description "My security group"   --vpc-id vpc-xxxxxxxx
```

**Salida de ejemplo / Example output**
```json
{
  "GroupId": "sg-01f4c77b81e9dc434"
}
```

**Listar SG / List SG**
```bash
aws ec2 describe-security-groups --group-ids sg-01f4c77b81e9dc434
```

### Reglas de Ingreso / Ingress Rules

**(Opcional/Optional) Detectar tu IP pública / Get your public IP**
```bash
curl https://checkip.amazonaws.com
# e.g. 186.96.109.58
```

> **Seguridad / Security:** Evita `0.0.0.0/0` en producción. Limita a tu IP cuando sea posible / Avoid `0.0.0.0/0` in production. Prefer your own IP when possible.

**Permitir SSH (22) a todo el mundo (demo) / Allow SSH (22) from anywhere (demo)**
```bash
aws ec2 authorize-security-group-ingress   --group-id sg-01f4c77b81e9dc434   --protocol tcp --port 22   --cidr 0.0.0.0/0
```

**Permitir RDP (3389) si usas Windows / Allow RDP (3389) if you use Windows**
```bash
aws ec2 authorize-security-group-ingress   --group-id sg-01f4c77b81e9dc434   --protocol tcp --port 3389   --cidr 0.0.0.0/0
```

---

## 🖥️ Paso 3 / Step 3 — Crear la Instancia / Launch the Instance

> **AMI y Subred / AMI & Subnet:** Necesitas un **AMI** válido para tu región y una **Subnet**. A continuación hay utilidades para descubrirlos.

**Descubrir Subnets / Discover Subnets**
```bash
aws ec2 describe-subnets   --query 'Subnets[].{SubnetId:SubnetId,Az:AvailabilityZone,VpcId:VpcId}'   --output table
```

**(Opcional recomendado / Recommended) Obtener el AMI más reciente de Amazon Linux 2 vía SSM**
```bash
aws ssm get-parameters   --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64   --query 'Parameters[0].Value' --output text
# Devuelve un ami-xxxxxxxx por región / Returns an ami-xxxxxxxx per region
```

**Ejecutar (ejemplo con AMI provista en el taller) / Run (example with provided AMI)**
```bash
aws ec2 run-instances   --image-id ami-032930428bf1abbff   --count 1   --instance-type t2.micro   --key-name MyKeyPair   --security-group-ids sg-01f4c77b81e9dc434   --subnet-id subnet-1175cf1d   --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=aws-cli-workshop}]'
```


<img width="941" height="718" alt="Screenshot 2025-09-17 092733" src="https://github.com/user-attachments/assets/cb845658-cc97-4a48-bc69-20ee115b2f88" /> *(Salida de `run-instances` mostrando InstanceId y estado / `run-instances` output with InstanceId and state)*

---

## 🔗 Paso 4 / Step 4 — Conectarse a la Instancia / Connect to the Instance

**Obtener DNS público / Get Public DNS**
```bash
aws ec2 describe-instances   --filters "Name=tag:Name,Values=aws-cli-workshop"   --query "Reservations[].Instances[].PublicDnsName"   --output text
# e.g. ec2-34-204-197-22.compute-1.amazonaws.com
```

**Conexión SSH (Amazon Linux / ec2-user):**
```bash
ssh -i "MyKeyPair.pem" ec2-user@ec2-34-204-197-22.compute-1.amazonaws.com
```

> **Windows + PuTTY:** Convierte `.pem` a `.ppk` con PuTTYgen y conecta como `ec2-user`.


<img width="937" height="707" alt="Screenshot 2025-09-17 092842" src="https://github.com/user-attachments/assets/fb3fa8c3-ab43-4861-beac-c9b7a7e9c0fa" /> *(Conexión SSH exitosa / Successful SSH session)*

---

## 📋 Paso 5 / Step 5 — Listar tus Instancias / List Your Instances

```bash
aws ec2 describe-instances   --filters "Name=instance-type,Values=t2.micro"   --query "Reservations[].Instances[].InstanceId"   --output table
```

**(Extra) Listar junto con estado, nombre y DNS / Extra: list with state, name, and DNS**
```bash
aws ec2 describe-instances   --query "Reservations[].Instances[].{Id:InstanceId,State:State.Name,Name:Tags[?Key=='Name']|[0].Value,DNS:PublicDnsName}"   --output table
```

<img width="945" height="897" alt="Screenshot 2025-09-17 093049" src="https://github.com/user-attachments/assets/ab6d7043-b9c4-47d3-8f7c-c2a54b0a1d24" /> *(Tabla de instancias / Instances table)*

<img width="932" height="901" alt="Screenshot 2025-09-17 093107" src="https://github.com/user-attachments/assets/404d329c-48c9-47a8-a90a-464621ac2128" />

<img width="944" height="247" alt="Screenshot 2025-09-17 093410" src="https://github.com/user-attachments/assets/8ad4c779-9e08-4202-82c1-e53d334973e8" />

---

## 🧹 Paso 6 / Step 6 — Limpieza / Clean Up

<img width="944" height="956" alt="Screenshot 2025-09-17 093523" src="https://github.com/user-attachments/assets/862c1da7-750d-4192-a615-0471da591524" />

<img width="765" height="575" alt="Screenshot 2025-09-17 103439" src="https://github.com/user-attachments/assets/2bd23a54-6a5e-444c-a078-b10c8c805dba" />

**Terminar la instancia / Terminate the instance**
```bash
aws ec2 terminate-instances --instance-ids i-07d0ddb36ea3e65a4
```

**Eliminar el Security Group / Delete the Security Group**
```bash
aws ec2 delete-security-group --group-id sg-903004f8
```

**Eliminar el Key Pair / Delete the Key Pair**
```bash
aws ec2 delete-key-pair --key-name MyKeyPair
rm -f MyKeyPair.pem
```

> **Orden sugerido / Suggested order:** 1) Terminate instance → 2) Delete SG → 3) Delete key pair.  
> Si no recuerdas los IDs / If you don’t remember IDs:
```bash
aws ec2 describe-instances --query "Reservations[].Instances[].{Id:InstanceId,State:State.Name,Name:Tags[?Key=='Name']|[0].Value}" --output table
aws ec2 describe-security-groups --query "SecurityGroups[].{Id:GroupId,Name:GroupName}" --output table
```

<img width="935" height="140" alt="Screenshot 2025-09-17 103814" src="https://github.com/user-attachments/assets/ed08123b-311f-44f9-b489-aa8058056463" />
<img width="914" height="570" alt="Screenshot 2025-09-17 103930" src="https://github.com/user-attachments/assets/13e22d97-eddb-4768-89d9-e93035f8c7e6" />
<img width="420" height="106" alt="Screenshot 2025-09-17 103934" src="https://github.com/user-attachments/assets/a3c067e2-1727-44fd-a0a0-f8bd60514104" />
<img width="939" height="360" alt="Screenshot 2025-09-17 104005" src="https://github.com/user-attachments/assets/224b608c-696a-4338-bee4-068d89601b0e" />

 *(Evidencia de recursos eliminados / Evidence of deleted resources)*

---

## 🛡️ Buenas Prácticas / Best Practices

- **Menos es más / Least privilege:** Usa políticas mínimas necesarias.
- **Restringe puertos / Restrict ports:** Prefiere tu IP, evita `0.0.0.0/0`.
- **Etiquetas / Tags:** Etiqueta todo para auditoría y costos.
- **Costos / Costs:** Termina recursos al finalizar.
- **Región consistente / Consistent region:** Mantén una región durante el taller.

---

## 🧰 Solución de Problemas / Troubleshooting

- **`InvalidClientTokenId` o `AuthFailure`:** Revisa Access Key/Secret en `~/.aws/credentials`. Asegúrate de que la cuenta/usuario existen y no estén inactivos.
- **`UnauthorizedOperation`:** Falta de permisos IAM. Solicita/ajusta política con acciones de EC2/VPC/SSM `Describe*`, `Create*`, `RunInstances`, etc.
- **`An error occurred (DryRunOperation)`:** Quitaste `--dry-run`? Si está presente y falla con `DryRunOperation`, significa que **sí** tienes permisos.
- **`InvalidAMIID.NotFound`:** El AMI no existe en tu región. Obtén uno válido vía SSM (sección Paso 3).
- **`Key is invalid format` o `UNPROTECTED PRIVATE KEY FILE`:** Revisa permisos del `.pem` (ver Paso 1).
- **No conecta por SSH:** Verifica:
  - SG permite **puerto 22** desde tu IP
  - Instancia tiene **Public IP** y está en una **Subnet pública** o con **EIP**
  - Usuario correcto (`ec2-user` en Amazon Linux 2)
- **Eliminar SG falla (`DependencyViolation`)**: Asegúrate de que **ninguna** instancia o ENI lo use.

---

## 📎 Apéndices / Appendices

### A. Descubrir AMI oficiales (variantes)
```bash
# Amazon Linux 2023 (x86_64) vía SSM
aws ssm get-parameters   --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64   --query 'Parameters[0].Value' --output text
```

### B. Comandos con salida tabular / Table-friendly outputs
```bash
aws ec2 describe-instances   --query "Reservations[].Instances[].{Id:InstanceId,Type:InstanceType,State:State.Name,AZ:Placement.AvailabilityZone}"   --output table
```

### C. Etiquetado / Tagging
```bash
aws ec2 create-tags   --resources i-0123456789abcdef0   --tags Key=Owner,Value=Daniel Key=Env,Value=Workshop
```

---

## 🏁 Conclusión / Conclusion

**ES:** ¡Felicidades! Has automatizado el ciclo de vida básico de una instancia EC2 con AWS CLI: creación, conexión, listado y limpieza.  
**EN:** Congratulations! You automated the basic EC2 lifecycle with AWS CLI: create, connect, list, and clean up.

---

## 📚 Referencias / References

- **AWS CLI User Guide**  
  EN: https://docs.aws.amazon.com/cli/latest/userguide/  
  ES: https://docs.aws.amazon.com/es_es/cli/latest/userguide/
- **Amazon EC2 User Guide (Linux Instances)**  
  https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/  
- **Systems Manager Parameter Store — Public AMIs**  
  https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-public-parameters-ami.html
- **EC2 Key Pairs**  
  https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html
- **Security Groups**  
  https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html

---
