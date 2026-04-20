ecs-logs-toggle/
│
├── lambda_function.py
└── templates/
    └── dashboard.html


# WEBHOOK_URL > add in env f lambda 

# add in configration :


Triggers 
    
    Application Load Balancer: SMSDevelopInternalLoadBalancer
    arn:aws:elasticloadbalancing:eu-west-3:905418298875:targetgroup/ecs-dashboard-lambda/cd157f91d1e4b71a
    Details
    Application Load Balancer: arn:aws:elasticloadbalancing:eu-west-3:905418298875:loadbalancer/app/SMSDevelopInternalLoadBalancer/cae26c632460cfcf
    isComplexStatement: No
    Service principal: elasticloadbalancing.amazonaws.com
    Statement ID: AllowExecutionFromALB-ecs-dashboard-lambda


    roles: 
      > Role name  : ecs-logs-toggle-role-ja42rk9a 
         1 > AmazonECS_FullAccess
         2 > ECS   
    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecs:ListServices",
                "ecs:DescribeServices",
                "ecs:DescribeTaskDefinition",
                "ecs:RegisterTaskDefinition",
                "ecs:UpdateService",
                "logs:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::905418298875:role/ecsTaskExecutionRole"
        }
    ]
}


        3 > AWSLambdaBasicExecutionRole-6366e9a0-853d-49a1-ba74-282dfd3488f3

           {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:eu-west-3:905418298875:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:eu-west-3:905418298875:log-group:/aws/lambda/ecs-logs-toggle:*"
            ]
        }
    ]
}





Resource summary :   AWS App Mesh   (5 actions, 1 resource )



Policy :

           1- FunctionURLAllowPublicAccess
             {
  "StringEquals": {
    "lambda:FunctionUrlAuthType": "NONE"
  }
}


           2- FunctionURLAllowInvokeAction
              {
  "Bool": {
    "lambda:InvokedViaFunctionUrl": "true"
  }
}


            3- AllowExecutionFromALB2

                Conditions
                None
           
           4- AllowExecutionFromALB-ecs-dashboard-lambda
          {
  "ArnLike": {
    "AWS:SourceArn": "arn:aws:elasticloadbalancing:eu-west-3:905418298875:targetgroup/ecs-dashboard-lambda/cd157f91d1e4b71a"
  }
}


in keycloak : b3l create user w b7ot uel lambda 



in lb: 
create target-group : 
    Registered target
        Lambda function  :  ecs-logs-toggle
 
create listner :  ( 3shan y login b keycloak )
     
        HTTPS:443
        
        Authenticate using OIDC
        
        Issuer: https://keycloak.qcingress-public.k8s.cequens.net/realms/master
        Token endpoint: https://keycloak.qcingress-public.k8s.cequens.net/realms/master/protocol/openid-connect/token
        User info endpoint: https://keycloak.qcingress-public.k8s.cequens.net/realms/master/protocol/openid-connect/userinfo
        Authorization endpoint: https://keycloak.qcingress-public.k8s.cequens.net/realms/master/protocol/openid-connect/auth
        Session cookie name: AWSELBAuthSessionCookie
        On unauthenticated: authenticate
        Scope: openid
        Forward to target group
        
        ecs-dashboard-lambda : 1 (100%)
        
        Target group stickiness: Off
        
        1 rule
        ARN
        ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09
        ecs-qc-dashboard.cequens.net (Certificate ID: ef9fdd04-87a3-49a9-a501-1417b2e6692c) 
        Off
        Not applicable
        Not applicable
        0 tags



in route 53 :   ( 3shan a3ml domain )
record-name : ecs-qc-dashboard.cequens.net
type : CNAME
value : LB-URL 

------------------------------------------------------------------------------
```bash
# lambda_function.py
import boto3
import json
import logging
import urllib.request  
import base64
import os

# Configure logging
logging.basicConfig(level=logging.ERROR)

ecs = boto3.client('ecs', region_name='eu-west-3')

# Configured clusters
CLUSTERS = [
    "sms-develop",
    "sms-utilities",
    "sms-mo-develop",
    "sms-provider-zong",
    "sms-provider-ufone",
    "sms-provider-telenour"
]

WEBHOOK_URL = os.environ.get("WEBHOOK_URL")


def extract_username(event):
    """Extracts username from JWT claims or headers."""
    try:
        authorizer = event.get('requestContext', {}).get('authorizer', {})
        claims = authorizer.get('jwt', {}).get('claims') or authorizer.get('claims') or authorizer
        username = claims.get('preferred_username') or claims.get('email') or claims.get('name') or claims.get('principalId')

        if username and str(username).strip() != 'Unknown':
            return str(username)

        headers = event.get('headers', {})
        if headers:
            headers_lower = {k.lower(): v for k, v in headers.items()}
            oidc_data = headers_lower.get('x-amzn-oidc-data')
            if oidc_data:
                payload = json.loads(base64.urlsafe_b64decode(oidc_data.split('.')[1] + '==').decode('utf-8'))
                username = payload.get('preferred_username') or payload.get('email') or payload.get('name')
                if username:
                    return str(username)

            username = headers_lower.get('x-forwarded-user') or headers_lower.get('x-email') or headers_lower.get('x-webauth-user')
            if username:
                return str(username)

    except Exception as e:
        logging.error(f"Error extracting username: {e}")

    return "Unknown"

def send_teams_notification(username, action, results):
    """Sends a notification to a Teams webhook."""
    lines = []
    for r in results:
        cluster_info = f" ({r['cluster']})" if "cluster" in r else ""
        if r["status"] != "error":
            lines.append(f"• {r['service']}{cluster_info} → {action} ✅")
        else:
            lines.append(f"• {r['service']}{cluster_info} → Error: {r.get('error','Unknown error')} ❌")

    message_text = (
        f"User **{username}** performed **{action.upper()}** on services:\n\n"
        + "\n\n".join(lines)
    )

    message = json.dumps({"text": message_text}).encode('utf-8')

    try:
        req = urllib.request.Request(
            WEBHOOK_URL,
            data=message,
            headers={'Content-Type': 'application/json'},
            method='POST'
        )
        with urllib.request.urlopen(req) as response:
            response.read()
    except Exception as e:
        logging.error(f"Failed to send Teams notification: {str(e)}")

def render_template(template_name, context):
    """Reads an HTML file and replaces placeholders with context values."""
    # Look for the template in the 'templates' folder relative to this script
    template_path = os.path.join(os.path.dirname(__file__), 'templates', template_name)
    with open(template_path, 'r', encoding='utf-8') as f:
        content = f.read()
    
    for key, value in context.items():
        placeholder = '{{' + key + '}}'
        content = content.replace(placeholder, str(value))
    
    return content

def lambda_handler(event, context):
    method = event.get('requestContext', {}).get('http', {}).get('method')
    if not method:
        method = event.get('httpMethod', 'GET')

    backend_username = extract_username(event)

    if method == "GET":
        services_desc = []
        for cluster in CLUSTERS:
            services_arns = []
            next_token = None

            while True:
                if next_token:
                    response = ecs.list_services(cluster=cluster, nextToken=next_token)
                else:
                    response = ecs.list_services(cluster=cluster)

                services_arns.extend(response['serviceArns'])

                next_token = response.get('nextToken')
                if not next_token:
                    break
            
            service_names = [arn.split('/')[-1] for arn in services_arns]
            if not service_names:
                continue

            for i in range(0, len(service_names), 10):
                batch = service_names[i:i+10]
                batch_data = ecs.describe_services(cluster=cluster, services=batch)['services']
                for svc in batch_data:
                    svc['clusterName'] = cluster
                services_desc.extend(batch_data)

        total_services = len(services_desc)
        running_count = sum(1 for svc in services_desc if svc['runningCount'] > 0)
        stopped_count = total_services - running_count

        service_html = ""
        for svc in services_desc:
            name = svc['serviceName']
            cluster = svc['clusterName']
            running = svc['runningCount']
            status_text = "Running" if running > 0 else "Stopped"
            status_class = "running" if running > 0 else "stopped"
            id_val = f"cb-{cluster}-{name}"
            service_html += f"""
            <div class="service-item" data-name="{name}" data-status="{status_text}">
                <div class="col-checkbox">
                    <input type="checkbox" class="service-checkbox" value="{cluster}:{name}" id="{id_val}">
                </div>
                <div class="col-name"><label for="{id_val}">{name} <small style="color:#64748b;">({cluster})</small></label></div>
                <div class="col-status"><span class="badge {status_class}">{status_text}</span></div>
            </div>
            """

        html_context = {
            'backend_username': backend_username,
            'total_services': total_services,
            'running_count': running_count,
            'stopped_count': stopped_count,
            'service_html': service_html
        }
        
        # Load dashboard.html from the templates/ folder
        html_response = render_template('dashboard.html', html_context)

        return {
            "statusCode": 200,
            "headers": {
                "Content-Type": "text/html",
                "Access-Control-Allow-Origin": "*"
            },
            "body": html_response
        }

    if method == "POST":
        body = json.loads(event.get('body','{}'))
        services_input = body.get('services', [])
        action = body.get('action','Disable')
        results, cluster_groups = [], {}
        for entry in services_input:
            if ':' in entry:
                c, s = entry.split(':', 1)
                if c not in cluster_groups: cluster_groups[c] = []
                cluster_groups[c].append(s)
        
        for cluster, services in cluster_groups.items():
            for i in range(0,len(services), 10):
                batch_desc = ecs.describe_services(cluster=cluster, services=services[i:i+10])['services']
                for svc_desc in batch_desc:
                    service = svc_desc['serviceName']
                    try:
                        task_def_arn = svc_desc['taskDefinition']
                        task_def = ecs.describe_task_definition(taskDefinition=task_def_arn)['taskDefinition']
                        if action.lower()=="stop":
                            ecs.update_service(cluster=cluster, service=service, desiredCount=0)
                        elif action.lower()=="start":
                            ecs.update_service(cluster=cluster, service=service, desiredCount=1)
                        elif action.lower() in ["enable", "disable"]:
                            for c in task_def['containerDefinitions']:
                                if action.lower()=="disable": c.pop("logConfiguration",None)
                                if action.lower()=="enable":
                                    c["logConfiguration"]={
                                        "logDriver": "awslogs",
                                        "options": {
                                            "awslogs-group":f"/ecs/{service}",
                                            "awslogs-region": "eu-west-3",
                                            "awslogs-stream-prefix": "ecs"
                                        }
                                    }
                            new_td = {k:v for k,v in task_def.items() if k not in [
                                "taskDefinitionArn", "revision", "status", "requiresAttributes", 
                                "compatibilities", "registeredAt", "registeredBy"
                            ]}
                            task_def_arn = ecs.register_task_definition(**new_td)['taskDefinition']['taskDefinitionArn']

                        ecs.update_service(cluster=cluster, service=service, taskDefinition=task_def_arn, forceNewDeployment=True)
                        results.append({"service":service, "cluster":cluster, "status":action})
                    except Exception as e:
                        results.append({"service":service, "cluster":cluster, "status": "error", "error":str(e)})

        send_teams_notification(backend_username, action.capitalize(), results)
        return {
            "statusCode": 200,
            "headers": {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*"
            },
            "body":json.dumps(results)
        }
```

```bash
<!-- templates/dashboard.html -->
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>QC ECS Dashboard</title>
<link href="https://fonts.googleapis.com/css2?family=Roboto:wght@400;500;700&display=swap" rel="stylesheet">
<style>
html, body { height: 100%; margin: 0; padding: 0; overflow: hidden; }
body { font-family: 'Roboto', sans-serif; background:#f0f4f8; color: #1e293b; display: flex; flex-direction: column; }
#dashboard-page { display: flex; flex-direction: column; height: 100%; }
header {
    background:#1e293b; color:white; padding: 15px 30px; font-size: 22px; font-weight:bold;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); flex-shrink: 0; z-index: 10; 
    display: grid; grid-template-columns: 1fr auto 1fr; align-items: center;
}
.header-title { grid-column: 2; text-align: center; }
.user-profile {
    grid-column: 3; justify-self: end; font-size: 14px; font-weight: 500; display: flex; align-items: center; gap: 8px; 
    background: rgba(255, 255, 255, 0.15); padding: 6px 14px; border-radius: 20px;
}
main {
    flex: 1; max-width: 1200px; width: 100%; margin: 15px auto; background:white; 
    padding: 20px 30px; border-radius: 12px; box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1); 
    display: flex; flex-direction: column; box-sizing: border-box; overflow: hidden;
}
h2 { margin-top: 0; font-size: 20px; margin-bottom: 20px; flex-shrink: 0; color: #1e293b; text-align: center; }
.layout-wrapper { display: flex; flex-direction: row; gap: 20px; flex: 1; min-height: 0; }
.left-panel { flex: 2; display: flex; flex-direction: column; min-width: 0; }
.right-panel { flex: 1; display: flex; flex-direction: column; background: #f8fafc; border-radius: 8px; border: 1px solid #e2e8f0; padding: 15px; min-width: 300px; }
.right-panel h3 { margin-top: 0; color: #1e293b; font-size: 16px; border-bottom: 1px solid #e2e8f0; padding-bottom: 10px; margin-bottom: 10px; text-align: center; }
.stats { display:flex; justify-content:center; gap: 20px; margin-bottom: 15px; font-weight:bold; background: #f8fafc; padding: 12px; border-radius: 8px; border: 1px solid #e2e8f0; flex-shrink: 0; }
.stats span { font-size: 14px; display:flex; align-items:center; }
.total { color:#64748b; }
.running { color:#22c55e; }
.stopped { color:#ef4444; }
.selected-count { color:#3b82f6; }
.controls-top { display:flex; flex-direction: column; gap: 10px; margin-bottom: 15px; flex-shrink: 0; }
input[type="search"] { width: 100%; padding: 10px 15px; font-size: 14px; border-radius: 8px; border: 1px solid #cbd5e1; box-sizing:border-box; transition: 0.2s; outline:none; }
input[type="search"]:focus { border-color: #3b82f6; box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.2); }
.select-all-container { font-weight: 500; display: flex; align-items: center; margin-top: 5px; }
.select-all-container input { margin-right: 10px; transform: scale(1.2); cursor: pointer; }
.list-container { flex-grow: 1; display:flex; flex-direction:column; min-height: 0; margin-bottom: 15px; border: 1px solid #cbd5e1; border-radius: 8px; background:#f9fafb; overflow:hidden; }
.list-header { display:flex; padding: 12px 14px; border-bottom: 2px solid #e2e8f0; font-weight: 700; color: #475569; font-size: 12px; text-transform: uppercase; letter-spacing: 0.5px; background: #f1f5f9; }
.col-checkbox { width: 35px; display: flex; align-items: center; }
.col-checkbox input { transform: scale(1.2); cursor: pointer; margin: 0; }
.col-name { flex: 1; cursor: pointer; user-select: none; display: flex; align-items: center; justify-content: flex-start; gap: 5px; }
.col-status { width: 100px; text-align: right; cursor: pointer; user-select: none; display: flex; align-items: center; justify-content: flex-end; gap: 5px; }
.service-list { overflow-y:auto; padding: 5px 0; flex-grow: 1; scrollbar-width: thin; }
.service-item { display:flex; align-items:center; padding: 8px 14px; border-bottom: 1px solid #f1f5f9; transition: 0.2s; }
.badge { padding: 4px 10px; border-radius: 20px; font-size: 12px; font-weight:bold; }
.badge.running { background:#dcfce7; color:#16a34a; }
.badge.stopped { background:#fee2e2; color:#dc2626; }
.actions-container { display:flex; justify-content:center; gap: 20px; flex-wrap:wrap; flex-shrink: 0; }
.action-group { background:#f8fafc; padding: 15px; border-radius: 10px; box-shadow: 0 3px 8px rgba(0, 0, 0, 0.08); text-align:center; flex: 1; min-width: 200px; }
.action-group h3 { margin-top: 0; margin-bottom: 12px; font-size: 16px; color:#1e293b; }
.action-group .btn-row { display: flex; gap: 10px; justify-content: center; flex-wrap: wrap; }
button.action-btn {
    padding: 10px 20px; font-size: 14px; border-radius: 10px; border:none; cursor:pointer; 
    font-weight: 600; box-shadow: 0 4px 6px rgba(0, 0, 0, 0.08);
    transition: all 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
}
button.action-btn:hover:not(:disabled) { transform: translateY(-2px) scale(1.04); box-shadow: 0 8px 15px rgba(0, 0, 0, 0.12); filter: brightness(1.05) saturate(1.05); }
button.action-btn:active:not(:disabled) { transform: translateY(0) scale(0.97); box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1); filter: brightness(0.9); }
button.enable { background-color:#3b82f6; color:white; }
button.disable { background-color:#64748b; color:white; }
button.restart { background-color:#f59e0b; color:white; }
button.start { background-color:#22c55e; color:white; }
button.stop { background-color:#b91c1c; color:white; }
button.action-btn:disabled { opacity: 0.5; cursor: not-allowed !important; transform: none !important; box-shadow: none !important; filter: none !important; }
#loading { display:none; text-align:center; margin-bottom: 10px; font-weight:bold; color:#3b82f6; font-size: 14px; flex-shrink: 0; }
#logsBox { background:#f1f5f9; padding: 15px; border-radius: 8px; flex-grow: 1; overflow-y:auto; font-family:monospace; font-size: 13px; color: #334155; border: 1px solid #cbd5e1; word-break: break-all; white-space: pre-wrap; }
</style>
</head>
<body>
<div id="dashboard-page">
    <header>
        <div style="grid-column: 1;"></div>
        <div class="header-title"> QC ECS Dashboard </div>
        <div class="user-profile" title="Logged in securely">👤 <span>{{backend_username}}</span></div>
    </header>
    <main>
    <h2>Manage Services</h2>
    <div class="layout-wrapper">
        <div class="left-panel">
            <div class="stats">
                <span class="total">Total: {{total_services}}</span>
                <span class="running">Running: {{running_count}}</span>
                <span class="stopped">Stopped: {{stopped_count}}</span>
                <span class="selected-count">Selected: 0</span>
            </div>
            <div class="controls-top">
                <input type="search" id="search" placeholder="Search services or clusters...">
                <div class="select-all-container">
                    <input type="checkbox" id="selectAll">
                    <label for="selectAll" style="cursor:pointer; font-size: 14px; color: #1e293b;">Select All Services</label>
                </div>
            </div>
            <div class="list-container">
                <div class="list-header">
                    <div class="col-checkbox"></div>
                    <div class="col-name" onclick="sortList('name')" id="sortName"> SERVICE NAME <span class="sort-icon">▼</span> </div>
                    <div class="col-status" onclick="sortList('status')" id="sortStatus"> STATUS <span class="sort-icon">▼</span> </div>
                </div>
                <div class="service-list" id="serviceList">{{service_html}}</div>
            </div>
            <div class="actions-container">
                <div class="action-group">
                    <h3>Service Control</h3>
                    <div class="btn-row">
                        <button class="action-btn restart" onclick="toggleAction('Restart')" disabled>Restart</button>
                        <button class="action-btn start" onclick="toggleAction('Start')" disabled>Start</button>
                        <button class="action-btn stop" onclick="toggleAction('Stop')" disabled>Stop</button>
                    </div>
                </div>
                <div class="action-group">
                    <h3>Logs Management</h3>
                    <div class="btn-row">
                        <button class="action-btn enable" onclick="toggleAction('Enable')" disabled>Enable Logs</button>
                        <button class="action-btn disable" onclick="toggleAction('Disable')" disabled>Disable Logs</button>
                    </div>
                </div>
            </div>
        </div>
        <div class="right-panel">
            <h3>Action Logs</h3>
            <div id="loading">Processing... Please wait ⏳</div>
            <div id="logsBox">Logs will appear here.</div>
        </div>
    </div>
    </main>
</div>
<script>
const checkboxes = () => Array.from(document.querySelectorAll('.service-checkbox'));
const updateButtons = () => {
    const selected = checkboxes().filter(cb => cb.checked);
    document.querySelectorAll('.actions-container button').forEach(btn => btn.disabled = selected.length === 0);
    document.querySelector('.selected-count').innerText = `Selected: ${selected.length}`;
};
document.getElementById('serviceList').addEventListener('change', () => updateButtons());
document.getElementById('selectAll').addEventListener('change', (event) => {
    checkboxes().forEach(cb => {
        const item = cb.closest('.service-item');
        if (item && item.style.display !== 'none') cb.checked = event.target.checked;
    });
    updateButtons();
});
document.getElementById('search').addEventListener('input', (e) => {
    const filter = e.target.value.toLowerCase();
    document.querySelectorAll('.service-item').forEach(item => {
        const name = item.getAttribute('data-name').toLowerCase();
        const cluster = item.querySelector('small')?.innerText.toLowerCase() || "";
        item.style.display = (name.includes(filter) || cluster.includes(filter)) ? 'flex' : 'none';
    });
    updateButtons();
});
async function toggleAction(action){
    const selected = checkboxes().filter(cb=>cb.checked).map(cb=>cb.value);
    if(selected.length===0) return;
    document.getElementById('loading').style.display = "block";
    document.getElementById('logsBox').innerText = "Executing action, please wait...";
    document.querySelectorAll('.actions-container button').forEach(b => b.disabled = true);
    try {
        const res = await fetch(window.location.href, {
            method:'POST',
            headers: { 'Content-Type':'application/json' },
            body:JSON.stringify({ services: selected, action: action })
        });
        const data = await res.json();
        let msg = "";
        if (Array.isArray(data)) {
            const successList = data.filter(i => i.status !== "error").map(i => `• ${i.service}`);
            const errorList = data.filter(i => i.status === "error").map(i => `❌ Error on ${i.service}: ${i.error || "Unknown error"}`);
            if (successList.length > 0) {
                msg += `✨ Done! The ${action} action on service name:\n` + successList.join("\n") + `\n\n🚀 Check ECS console to see the changes.`;
            }
            if (errorList.length > 0) {
                if (msg) msg += "\n\n" + "-".repeat(30) + "\n\n";
                msg += errorList.join("\n");
            }
            document.getElementById('logsBox').innerText = msg || "No services were updated.";
        } else {
            document.getElementById('logsBox').innerText = JSON.stringify(data, null, 2);
        }
    } catch(err) {
        document.getElementById('logsBox').innerText = "Error: " + err.message;
    } finally {
        document.getElementById('loading').style.display = "none"; 
        updateButtons();
    }
}
window.addEventListener('DOMContentLoaded', () => updateButtons());
</script>
</body>
</html>
```
