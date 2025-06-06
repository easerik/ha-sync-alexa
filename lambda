import json
import urllib.request
import urllib.parse
import os
import time
import base64
import boto3
from collections import defaultdict
import traceback
import logging

# ===============================================================================
# CONFIGURATION AND LOGGING
# ===============================================================================
# PT-BR: Configuração do logger para substituir o print() e integrar com CloudWatch.
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# PT-BR: Configuração inicial a partir das variáveis de ambiente.
HA_URL = os.environ.get('HA_URL')
HA_TOKEN = os.environ.get('HA_TOKEN')
WEBHOOK_SECRET = os.environ.get('WEBHOOK_SECRET')
ALEXA_CLIENT_ID = os.environ.get('ALEXA_CLIENT_ID')
ALEXA_CLIENT_SECRET = os.environ.get('ALEXA_CLIENT_SECRET')
DYNAMODB_TABLE_NAME = os.environ.get('DYNAMODB_TABLE', 'alexa-user-tokens')
HA_DISCOVERY_TAG = os.environ.get('HA_DISCOVERY_TAG')

# PT-BR: Clientes para serviços AWS e cache em memória.
dynamodb = boto3.resource('dynamodb')
tokens_table = dynamodb.Table(DYNAMODB_TABLE_NAME)
request_counts = defaultdict(list)


# ===============================================================================
# 🛡️ SECURITY MODULE
# ===============================================================================
def security_check(headers, source_ip):
	"""
	Performs multi-layered security checks for incoming webhooks.
	PT-BR: Realiza checagens de segurança em múltiplas camadas para webhooks.
	"""
	if not WEBHOOK_SECRET or headers.get('x-webhook-secret') != WEBHOOK_SECRET:
		logger.warning("BLOCKED: Invalid or missing webhook secret.")
		return False
	if headers.get('cf-connecting-ip') and not headers.get('cf-ray'):
		logger.warning("BLOCKED: Potential Cloudflare header spoofing.")
		return False
	
	real_ip = headers.get('cf-connecting-ip') or source_ip
	now = time.time()
	window, max_requests = 60, 100
	request_counts[real_ip] = [t for t in request_counts[real_ip] if now - t < window]
	if len(request_counts[real_ip]) >= max_requests:
		logger.warning(f"RATE LIMIT: IP {real_ip} exceeded request limit.")
		return False
	request_counts[real_ip].append(now)
	return True

# ===============================================================================
# TOKEN MANAGEMENT (DYNAMODB)
# ===============================================================================
def exchange_code_for_tokens(code):
	"""
	Exchanges an Alexa authorization code for access and refresh tokens.
	PT-BR: Troca um código de autorização da Alexa por tokens de acesso e de atualização.
	"""
	try:
		url = "https://api.amazon.com/auth/o2/token"
		payload = {"grant_type": "authorization_code", "code": code, "client_id": ALEXA_CLIENT_ID, "client_secret": ALEXA_CLIENT_SECRET}
		data = urllib.parse.urlencode(payload).encode('utf-8')
		req = urllib.request.Request(url, data=data, method='POST', headers={"Content-Type": "application/x-www-form-urlencoded"})
		with urllib.request.urlopen(req, timeout=10) as resp:
			token_data = json.loads(resp.read())
			if 'access_token' not in token_data:
				logger.error(f"Failed to exchange code, 'access_token' not in response: {token_data}")
				return None
			return {'access_token': token_data['access_token'], 'refresh_token': token_data.get('refresh_token'), 'expires_at': int(time.time()) + 3600}
	except Exception:
		logger.exception("Exception in exchange_code_for_tokens")
		return None

def refresh_user_token(user_id, refresh_token):
	"""
	Refreshes an expired access token using a refresh token.
	PT-BR: Atualiza um token de acesso expirado usando o refresh token.
	"""
	if not refresh_token:
		logger.warning(f"Refresh token not available for user {user_id}.")
		return None
	try:
		url = "https://api.amazon.com/auth/o2/token"
		payload = {"grant_type": "refresh_token", "refresh_token": refresh_token, "client_id": ALEXA_CLIENT_ID, "client_secret": ALEXA_CLIENT_SECRET}
		data = urllib.parse.urlencode(payload).encode('utf-8')
		req = urllib.request.Request(url, data=data, method='POST', headers={"Content-Type": "application/x-www-form-urlencoded"})
		with urllib.request.urlopen(req, timeout=10) as resp:
			token_data = json.loads(resp.read())
			if 'access_token' not in token_data:
				logger.error(f"Failed to refresh token, 'access_token' not in response: {token_data}")
				return None
			new_tokens = {':at': token_data['access_token'], ':rt': token_data.get('refresh_token', refresh_token), ':ea': int(time.time()) + 3600, ':ua': int(time.time())}
			tokens_table.update_item(Key={'user_id': user_id}, UpdateExpression='SET access_token = :at, refresh_token = :rt, expires_at = :ea, updated_at = :ua', ExpressionAttributeValues=new_tokens)
			logger.info(f"Successfully refreshed token for user {user_id}")
			return new_tokens[':at']
	except Exception:
		logger.exception(f"Exception in refresh_user_token for user {user_id}")
		return None

def get_user_access_token():
	"""
	Gets a valid user access token from DynamoDB, refreshing if necessary.
	PT-BR: Obtém um token de usuário válido do DynamoDB, atualizando se necessário.
	"""
	try:
		response = tokens_table.scan(Limit=1)
		if not (items := response.get('Items')):
			logger.warning("No user tokens found in DynamoDB.")
			return None
		user_data = items[0]
		if int(time.time()) < user_data.get('expires_at', 0) - 300: # 5 min buffer
			return user_data['access_token']
		logger.info(f"User token for {user_data.get('user_id')} expired, refreshing...")
		return refresh_user_token(user_data.get('user_id'), user_data.get('refresh_token'))
	except Exception:
		logger.exception("Exception in get_user_access_token")
		return None

# ===============================================================================
# CENTRALIZED API CALLER
# ===============================================================================
def _call_ha_api(endpoint, method='GET', json_payload=None):
	"""
	Centralized function to make API calls to Home Assistant.
	PT-BR: Função centralizada para fazer chamadas à API do Home Assistant.
	"""
	try:
		url = f'{HA_URL}/api/{endpoint}'
		data = json.dumps(json_payload).encode('utf-8') if json_payload else None
		req = urllib.request.Request(url, data=data, method=method, headers={'Authorization': f'Bearer {HA_TOKEN}', 'Content-Type': 'application/json'})
		with urllib.request.urlopen(req, timeout=7) as resp:
			if resp.status >= 300:
				logger.error(f"Home Assistant API returned status {resp.status} for {endpoint}")
				return None
			if resp.status == 204: # No Content
				return {} # Success with no data to return
			return json.loads(resp.read().decode('utf-8'))
	except Exception:
		logger.exception(f"Exception calling Home Assistant API endpoint '{endpoint}'")
		return None

# ===============================================================================
# ALEXA DIRECTIVE HANDLERS (Alexa -> Home Assistant)
# ===============================================================================
def handle_accept_grant(event):
	"""
	Handles the AcceptGrant directive for account linking.
	PT-BR: Lida com a diretiva AcceptGrant para vincular a conta do usuário.
	"""
	try:
		payload = event.get('directive', {}).get('payload', {})
		code = payload.get('grant', {}).get('code')
		user_id = payload.get('grantee', {}).get('token')
		if not code or not user_id:
			return create_error_response(event, "INVALID_AUTHORIZATION_CREDENTIAL", "Missing grant code or grantee token.")
		if not (user_tokens := exchange_code_for_tokens(code)):
			return create_error_response(event, "INVALID_AUTHORIZATION_CREDENTIAL", "Failed to exchange code for tokens")
		tokens_table.put_item(Item={'user_id': user_id, 'created_at': int(time.time()), 'updated_at': int(time.time()), **user_tokens})
		return {"event": {"header": {"namespace": "Alexa.Authorization", "name": "AcceptGrant.Response", "payloadVersion": "3", "messageId": "accept-grant-response"}, "payload": {}}}
	except Exception:
		logger.exception("Exception in handle_accept_grant")
		return create_error_response(event, "INTERNAL_ERROR", "Failed to process AcceptGrant.")

def handle_discovery(event):
	"""
	Handles the Discover directive to find devices in Home Assistant.
	PT-BR: Lida com a diretiva Discover para encontrar dispositivos no Home Assistant.
	"""
	endpoints = []
	try:
		ha_states = _call_ha_api("states")
		if ha_states is None: raise ConnectionError("Failed to fetch states from Home Assistant")
		for state in ha_states:
			if (endpoint := build_discovery_endpoint(state)):
				endpoints.append(endpoint)
		logger.info(f"Discovery found {len(endpoints)} devices.")
		return {"event": {"header": {"namespace": "Alexa.Discovery", "name": "Discover.Response", "payloadVersion": "3", "messageId": "discovery-response"}, "payload": {"endpoints": endpoints}}}
	except Exception:
		logger.exception("Exception in handle_discovery")
		return create_error_response(event, "INTERNAL_ERROR", "Discovery failed")

def handle_report_state(event):
	"""
	Handles the ReportState directive, fetching the current state from Home Assistant.
	PT-BR: Lida com a diretiva ReportState, buscando o estado atual do Home Assistant.
	"""
	endpoint = event.get('directive', {}).get('endpoint', {})
	entity_id = endpoint.get('endpointId')
	if not entity_id: return create_error_response(event, "INVALID_VALUE", "Missing endpointId.")
	try:
		ha_state = _call_ha_api(f"states/{entity_id}")
		if ha_state is None: raise ConnectionError(f"Could not retrieve state for {entity_id}")
		
		properties = build_alexa_properties(ha_state)
		header = event.get('directive', {}).get('header', {})
		return {
			"event": {"header": {"namespace": "Alexa", "name": "StateReport", "payloadVersion": "3", "messageId": header.get('messageId', 'msg') + "-R", "correlationToken": header.get('correlationToken')}, "endpoint": endpoint, "payload": {}},
			"context": {"properties": properties}
		}
	except Exception:
		logger.exception(f"Exception in handle_report_state for {entity_id}")
		return create_error_response(event, "ENDPOINT_UNREACHABLE", f"Could not retrieve state for {entity_id}")

def handle_power_control(event):
	"""
	Handles PowerController directives (TurnOn, TurnOff).
	PT-BR: Lida com diretivas PowerController (Ligar/Desligar).
	"""
	endpoint = event.get('directive', {}).get('endpoint', {})
	entity_id = endpoint.get('endpointId')
	if not entity_id: return create_error_response(event, "INVALID_VALUE", "Missing endpointId.")
	
	command = "turn_on" if event['directive']['header']['name'] == 'TurnOn' else "turn_off"
	domain = entity_id.split('.')[0]
	
	if _call_ha_api(f"services/{domain}/{command}", method='POST', json_payload={'entity_id': entity_id}) is not None:
		return build_control_response(event, "powerState", "ON" if command == "turn_on" else "OFF")
	return create_error_response(event, "DEPENDENT_SERVICE_UNAVAILABLE", "Failed to send command to Home Assistant")

def handle_brightness_control(event):
	"""
	Handles BrightnessController directives.
	PT-BR: Lida com diretivas de controle de brilho.
	"""
	endpoint = event.get('directive', {}).get('endpoint', {})
	entity_id = endpoint.get('endpointId')
	payload = event.get('directive', {}).get('payload', {})
	if not entity_id: return create_error_response(event, "INVALID_VALUE", "Missing endpointId.")
	
	ha_payload = {}
	alexa_brightness = payload.get('brightness')
	if event['directive']['header']['name'] == 'SetBrightness':
		if alexa_brightness is None: return create_error_response(event, "INVALID_VALUE", "Missing brightness value.")
		ha_payload['brightness'] = int(round(alexa_brightness * 2.55))
	elif event['directive']['header']['name'] == 'AdjustBrightness':
		ha_payload['brightness_step_pct'] = payload.get('brightnessDelta')
	
	if _call_ha_api("services/light/turn_on", method='POST', json_payload={'entity_id': entity_id, **ha_payload}) is not None:
		return build_control_response(event, "brightness", alexa_brightness)
	return create_error_response(event, "DEPENDENT_SERVICE_UNAVAILABLE", "Failed to set brightness")

def handle_color_control(event):
	"""
	Handles ColorController directives.
	PT-BR: Lida com diretivas de controle de cor.
	"""
	endpoint = event.get('directive', {}).get('endpoint', {})
	entity_id = endpoint.get('endpointId')
	color = event.get('directive', {}).get('payload', {}).get('color', {})
	if not entity_id or not color: return create_error_response(event, "INVALID_VALUE", "Missing endpointId or color payload.")
	
	ha_payload = {'hs_color': [color.get('hue', 0.0), color.get('saturation', 0.0) * 100.0], 'brightness_pct': int(color.get('brightness', 0.0) * 100.0)}
	if _call_ha_api("services/light/turn_on", method='POST', json_payload={'entity_id': entity_id, **ha_payload}) is not None:
		return build_control_response(event, "color", color)
	return create_error_response(event, "DEPENDENT_SERVICE_UNAVAILABLE", "Failed to set color")

def handle_color_temperature_control(event):
	"""
	Handles ColorTemperatureController directives.
	PT-BR: Lida com diretivas de controle de temperatura de cor.
	"""
	endpoint = event.get('directive', {}).get('endpoint', {})
	entity_id = endpoint.get('endpointId')
	kelvin = event.get('directive', {}).get('payload', {}).get('colorTemperatureInKelvin')
	if not entity_id or not kelvin: return create_error_response(event, "INVALID_VALUE", "Missing endpointId or kelvin value.")

	if _call_ha_api("services/light/turn_on", method='POST', json_payload={'entity_id': entity_id, 'kelvin': kelvin}) is not None:
		return build_control_response(event, "colorTemperatureInKelvin", kelvin)
	return create_error_response(event, "DEPENDENT_SERVICE_UNAVAILABLE", "Failed to set color temperature")

def handle_range_control(event):
	"""
	Handles RangeController directives (e.g., covers).
	PT-BR: Lida com diretivas RangeController (ex: persianas).
	"""
	endpoint = event.get('directive', {}).get('endpoint', {})
	entity_id = endpoint.get('endpointId')
	range_value = event.get('directive', {}).get('payload', {}).get('rangeValue')
	if not entity_id or range_value is None: return create_error_response(event, "INVALID_VALUE", "Missing endpointId or range value.")
	
	if _call_ha_api("services/cover/set_cover_position", method='POST', json_payload={'entity_id': entity_id, 'position': range_value}) is not None:
		return build_control_response(event, "rangeValue", range_value, instance="Cover.Position")
	return create_error_response(event, "DEPENDENT_SERVICE_UNAVAILABLE", "Failed to set range")

def handle_mode_control(event):
	"""
	Handles ModeController directives (e.g., cover open/close).
	PT-BR: Lida com diretivas ModeController (ex: abrir/fechar persiana).
	"""
	endpoint = event.get('directive', {}).get('endpoint', {})
	entity_id = endpoint.get('endpointId')
	mode = event.get('directive', {}).get('payload', {}).get('mode')
	if not entity_id or not mode: return create_error_response(event, "INVALID_VALUE", "Missing endpointId or mode value.")
	
	action = "open_cover" if mode == "Cover.Open" else "close_cover" if mode == "Cover.Closed" else None
	if action and _call_ha_api(f"services/cover/{action}", method='POST', json_payload={'entity_id': entity_id}) is not None:
		return build_control_response(event, "mode", mode, instance="Cover.State")
	return create_error_response(event, "DEPENDENT_SERVICE_UNAVAILABLE", "Failed to set mode")

def handle_script_activate(event):
	"""
	Handles SceneController directives to activate scripts.
	PT-BR: Lida com diretivas SceneController para ativar scripts.
	"""
	endpoint = event.get('directive', {}).get('endpoint', {})
	entity_id = endpoint.get('endpointId')
	if not entity_id: return create_error_response(event, "INVALID_VALUE", "Missing endpointId.")
	
	if _call_ha_api("services/script/turn_on", method='POST', json_payload={'entity_id': entity_id}) is not None:
		return build_control_response(event)
	return create_error_response(event, "DEPENDENT_SERVICE_UNAVAILABLE", "Failed to activate script")


# ===============================================================================
# WEBHOOK HANDLER (Home Assistant -> Alexa)
# ===============================================================================
def handle_change_report(headers, body_dict, source_ip):
	"""
	Processes webhooks from Home Assistant to send proactive ChangeReports to Alexa.
	PT-BR: Processa webhooks do HA para enviar ChangeReports proativos para a Alexa.
	"""
	try:
		if not security_check(headers, source_ip):
			return {"statusCode": 403, "body": json.dumps({"error": "Security validation failed"})}
		
		if not (entities := body_dict.get('entities', [])):
			logger.info("Webhook received with empty entities list.")
			return {"statusCode": 200, "body": json.dumps({"message": "Empty entities list"})}
		
		if not (user_access_token := get_user_access_token()):
			logger.error("User access token is not available for ChangeReport.")
			return {"statusCode": 503, "body": json.dumps({"error": "User access token unavailable."})}

		successful_sends = 0
		for entity in entities:
			if properties := build_alexa_properties(entity):
				report = build_change_report_payload(user_access_token, entity.get('entity_id'), properties)
				if send_to_alexa_gateway(user_access_token, report):
					successful_sends += 1
		
		logger.info(f"ChangeReport processed: {successful_sends}/{len(entities)} successful sends.")
		return {"statusCode": 200, "body": json.dumps({"successful_sends": successful_sends, "total_entities": len(entities)})}
	except Exception:
		logger.exception("FATAL ERROR in handle_change_report")
		return {"statusCode": 500, "body": json.dumps({"error": "Internal server error."})}

# ===============================================================================
# HELPER AND LOGIC FUNCTIONS
# ===============================================================================
DEVICE_CAPABILITIES = {
	"light": {
		"display_categories": ["LIGHT"],
		"capabilities": {
			"power": {"interface": "Alexa.PowerController", "properties": {"supported": [{"name": "powerState"}], "proactivelyReported": True, "retrievable": True}},
			"brightness": {"interface": "Alexa.BrightnessController", "properties": {"supported": [{"name": "brightness"}], "proactivelyReported": True, "retrievable": True}, "ha_check": lambda attrs: 'brightness' in attrs.get('supported_color_modes', []) or 'brightness' in attrs},
			"color": {"interface": "Alexa.ColorController", "properties": {"supported": [{"name": "color"}], "proactivelyReported": True, "retrievable": True}, "ha_check": lambda attrs: any(x in attrs.get('supported_color_modes', []) for x in ['hs', 'rgb'])},
			"color_temperature": {"interface": "Alexa.ColorTemperatureController", "properties": {"supported": [{"name": "colorTemperatureInKelvin"}], "proactivelyReported": True, "retrievable": True}, "ha_check": lambda attrs: 'color_temp' in attrs.get('supported_color_modes', [])}
		}
	},
	"cover": {
		"display_categories": ["BLIND"],
		"capabilities": {
			"range": {"interface": "Alexa.RangeController", "instance": "Cover.Position", "properties": {"supported": [{"name": "rangeValue"}], "proactivelyReported": True, "retrievable": True}, "configuration": {"supportedRange": {"minimumValue": 0, "maximumValue": 100}}},
			"mode": {"interface": "Alexa.ModeController", "instance": "Cover.State", "properties": {"supported": [{"name": "mode"}], "proactivelyReported": True, "retrievable": True}, "configuration": {"supportedModes": [{"value": "Cover.Open"}, {"value": "Cover.Closed"}]}}
		}
	},
	"script": {
		"display_categories": ["SCENE_TRIGGER"],
		"capabilities": {
			"scene": {"interface": "Alexa.SceneController", "version": "3", "supportsDeactivation": False, "proactivelyReported": False}
		}
	}
}

def build_discovery_endpoint(state):
	"""
	Builds an Alexa discovery endpoint using the centralized capability map.
	PT-BR: Constrói um endpoint da Alexa usando o mapa central de capacidades.
	"""
	attributes = state.get('attributes', {})
	if HA_DISCOVERY_TAG and not attributes.get(HA_DISCOVERY_TAG, False):
		return None
	
	entity_id = state.get('entity_id')
	if not entity_id: return None
	
	domain = entity_id.split('.')[0]
	if domain not in DEVICE_CAPABILITIES: return None

	domain_map = DEVICE_CAPABILITIES[domain]
	alexa_capabilities = [{"type": "AlexaInterface", "interface": "Alexa", "version": "3"}]

	for cap_key, cap_data in domain_map.get("capabilities", {}).items():
		if "ha_check" in cap_data and not cap_data["ha_check"](attributes):
			continue
		
		alexa_cap = {"type": "AlexaInterface", "version": "3", **cap_data}
		alexa_cap.pop("ha_check", None)
		alexa_capabilities.append(alexa_cap)

	if len(alexa_capabilities) <= 1: return None

	return {"endpointId": entity_id, "manufacturerName": "Home Assistant", "friendlyName": attributes.get('friendly_name', entity_id), "description": f"{domain} via HA", "displayCategories": domain_map["display_categories"], "capabilities": alexa_capabilities}

def build_control_response(event, prop_name=None, prop_value=None, instance=None):
	"""
	Builds a generic successful response for a control directive.
	PT-BR: Constrói uma resposta de sucesso genérica para uma diretiva de controle.
	"""
	header = event.get('directive', {}).get('header', {})
	endpoint = event.get('directive', {}).get('endpoint', {})
	if not header or not endpoint: return create_error_response(event, "INVALID_DIRECTIVE", "Malformed directive.")

	response = {"event": {"header": {"namespace": "Alexa", "name": "Response", "payloadVersion": "3", "messageId": f"{header.get('messageId', 'msg')}-R", "correlationToken": header.get('correlationToken')}, "endpoint": endpoint, "payload": {}}}
	
	if prop_name and prop_value is not None:
		prop_obj = {"namespace": header.get('namespace'), "name": prop_name, "value": prop_value, "timeOfSample": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()), "uncertaintyInMilliseconds": 500}
		if instance: prop_obj["instance"] = instance
		response["context"] = {"properties": [prop_obj]}
	
	if header.get('namespace') == 'Alexa.SceneController' and header.get('name') == 'Activate':
		response['event']['header'].update({'namespace': 'Alexa.SceneController', 'name': 'ActivationStarted'})
		response.pop('context', None)
	return response

def build_alexa_properties(entity_state):
	"""
	Builds a list of Alexa properties from a Home Assistant state object.
	PT-BR: Constrói uma lista de propriedades da Alexa a partir de um objeto de estado do HA.
	"""
	properties = []
	attributes = entity_state.get('attributes', {})
	entity_id, state_val, domain = entity_state.get('entity_id'), entity_state.get('state'), entity_state.get('entity_id', '').split('.')[0]
	if not entity_id: return []
	
	ts = time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
	
	if domain == 'light':
		properties.append({"namespace": "Alexa.PowerController", "name": "powerState", "value": "ON" if state_val == "on" else "OFF", "timeOfSample": ts, "uncertaintyInMilliseconds": 500})
		if (brightness := attributes.get('brightness')) is not None:
			properties.append({"namespace": "Alexa.BrightnessController", "name": "brightness", "value": int(round(float(brightness) / 2.55)), "timeOfSample": ts, "uncertaintyInMilliseconds": 500})
		if (hs_color := attributes.get('hs_color')) and isinstance(hs_color, (list, tuple)):
			alexa_color = {"hue": float(hs_color[0]), "saturation": float(hs_color[1]) / 100.0, "brightness": float(brightness or 0) / 255.0}
			properties.append({"namespace": "Alexa.ColorController", "name": "color", "value": alexa_color, "timeOfSample": ts, "uncertaintyInMilliseconds": 500})
		if (kelvin := attributes.get('color_temp_kelvin')) is not None:
			properties.append({"namespace": "Alexa.ColorTemperatureController", "name": "colorTemperatureInKelvin", "value": kelvin, "timeOfSample": ts, "uncertaintyInMilliseconds": 500})
	elif domain == 'cover':
		if (position := attributes.get('current_position')) is not None:
			properties.append({"namespace": "Alexa.RangeController", "instance": "Cover.Position", "name": "rangeValue", "value": position, "timeOfSample": ts, "uncertaintyInMilliseconds": 500})
	
	return properties

def build_change_report_payload(token, entity_id, properties):
	"""
	Builds the full ChangeReport payload.
	PT-BR: Monta o payload completo do ChangeReport.
	"""
	return {"event": {"header": {"namespace": "Alexa", "name": "ChangeReport", "payloadVersion": "3", "messageId": f"cr-{int(time.time()*1000)}"},
			"endpoint": {"scope": {"type": "BearerToken", "token": token}, "endpointId": entity_id},
			"payload": {"change": {"cause": {"type": "PHYSICAL_INTERACTION"}, "properties": properties}}}}

def send_to_alexa_gateway(token, payload):
	"""
	Sends a payload to the Alexa Event Gateway.
	PT-BR: Envia dados para o Alexa Event Gateway.
	"""
	try:
		data = json.dumps(payload).encode('utf-8')
		req = urllib.request.Request("https://api.amazonalexa.com/v3/events", data=data, method='POST', headers={'Authorization': f'Bearer {token}', 'Content-Type': 'application/json'})
		with urllib.request.urlopen(req, timeout=8) as response:
			return response.status == 202
	except Exception:
		logger.exception("Exception sending to Alexa Gateway")
		return False

def create_error_response(event, error_type, message):
	"""
	Creates a standard Alexa ErrorResponse.
	PT-BR: Cria uma resposta de erro padrão da Alexa.
	"""
	header = event.get('directive', {}).get('header', {})
	logger.error(f"Creating ErrorResponse: Type={error_type}, Message={message}")
	return {"event": {"header": {"namespace": "Alexa", "name": "ErrorResponse", "payloadVersion": "3", "messageId": f"{header.get('messageId', 'err')}-R", "correlationToken": header.get('correlationToken')},
			"endpoint": event.get('directive', {}).get('endpoint'), "payload": {"type": error_type, "message": message}}}

# ===============================================================================
# MAIN LAMBDA HANDLER AND ROUTER
# ===============================================================================
def lambda_handler(event, context):
	"""
	Main entry point that routes requests from Alexa and Home Assistant webhooks.
	PT-BR: Ponto de entrada principal que roteia requisições da Alexa e de webhooks do HA.
	"""
	logger.info("Lambda invoked.")

	if 'directive' in event:
		header = event.get('directive', {}).get('header', {})
		namespace = header.get('namespace')
		name = header.get('name')
		
		handler_map = {
			"Alexa.Authorization": handle_accept_grant,
			"Alexa.Discovery": handle_discovery,
			"Alexa": {"ReportState": handle_report_state},
			"Alexa.PowerController": handle_power_control,
			"Alexa.BrightnessController": handle_brightness_control,
			"Alexa.ColorController": handle_color_control,
			"Alexa.ColorTemperatureController": handle_color_temperature_control,
			"Alexa.RangeController": handle_range_control,
			"Alexa.ModeController": handle_mode_control,
			"Alexa.SceneController": handle_script_activate
		}

		handler = handler_map.get(namespace)
		if isinstance(handler, dict): handler = handler.get(name)

		if handler:
			return handler(event)
		
		return create_error_response(event, "INVALID_DIRECTIVE", f"The directive {namespace}.{name} is not supported.")

	if event.get('requestContext', {}).get('http', {}).get('method') == 'POST':
		source_ip = event.get('requestContext', {}).get('http', {}).get('sourceIp', 'unknown_ip')
		headers = event.get('headers', {})
		body_str = base64.b64decode(event['body']).decode('utf-8') if event.get('isBase64Encoded') else event.get('body', '{}')
		try:
			body_dict = json.loads(body_str)
		except json.JSONDecodeError:
			logger.error(f"Invalid JSON received in webhook body: {body_str[:200]}")
			return {"statusCode": 400, "body": json.dumps({"error": "Invalid JSON format."})}
			
		return handle_change_report(headers=headers, body_dict=body_dict, source_ip=source_ip)

	logger.warning(f"Unrecognized event received: {json.dumps(event, default=str)[:300]}")
	return {"statusCode": 400, "body": json.dumps({"error": "Unrecognized request format."})}
