import asyncio
import simpleobsws
from time import time


class OBSsync():
    def __init__(self):
        self.wsM = None
        self.wsS = None
        self.transform_data = None
        self.last_transform_update = time()

    async def connect_master(self, host, port, password=''):
        parameters = simpleobsws.IdentificationParameters()
        parameters.eventSubscriptions = sum(2**m for m in [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 19])
        self.wsM = simpleobsws.WebSocketClient(
            url=f'ws://localhost:4444',
            password='HAVILOG',
            identification_parameters=parameters)
        await self.wsM.connect()
        self.wsM.register_event_callback(self.on_scene_created)
        self.wsM.register_event_callback(self.on_scene_removed)
        self.wsM.register_event_callback(self.on_scene_selection_change)
        self.wsM.register_event_callback(
            self.on_scene_item_enable_state_changed)
        self.wsM.register_event_callback(self.on_scene_item_list_reindexed)
        self.wsM.register_event_callback(self.on_scene_item_lock_state_changed)
        self.wsM.register_event_callback(self.on_scene_item_selected)
        self.wsM.register_event_callback(self.on_scene_item_transform_changed)
        self.wsM.register_event_callback(self.on_input_created)
        self.wsM.register_event_callback(self.on_input_removed)
        # self.wsM.register_event_callback(self.test)
        print('Connection with "Master" established.\n')

    async def test(self, event_type, event_data):
        print(f'New event! Type: {event_type} | Raw Data: {event_data}')

    async def connect_slave(self, host, port, password=''):
        parameters = simpleobsws.IdentificationParameters()
        self.wsS = simpleobsws.WebSocketClient(
            url=f'ws://localhost:4445',
            password='HAVILOG',
            identification_parameters=parameters)
        await self.wsS.connect()
        print('Connection with "Slave" established.\n')

    async def on_scene_created(self, event_type, event_data):
        # {'sceneName': 'Сцена 2', 'isGroup': False}
        if event_type != 'SceneCreated':
            return
        print(f'Creating scene {event_data["sceneName"]}.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        print()
        await self.wsS.call(simpleobsws.Request('CreateScene', {
            'sceneName': event_data['sceneName']
        }))

    async def on_scene_removed(self, event_type, event_data):
        # {'sceneName': 'Сцена 2', 'isGroup': False}
        if event_type != 'SceneRemoved':
            return
        print(f'Removing scene {event_data["sceneName"]}.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        print()
        await self.wsS.call(simpleobsws.Request('RemoveScene', {
            'sceneName': event_data['sceneName']
        }))

    async def on_scene_selection_change(self, event_type, event_data):
        # {'sceneName': 'Сцена'}
        if event_type != 'CurrentProgramSceneChanged':
            return
        print(f'Scene changed to {event_data["sceneName"]}.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        scene = event_data['sceneName']
        result = await self.wsS.call(simpleobsws.Request('GetSceneList'))
        scenes = [m['sceneName'] for m in result.responseData['scenes']]
        if scene not in scenes:
            print(f'Scene {scene} not found in Slave.\n')
            return
        print()
        await self.wsS.call(simpleobsws.Request('SetCurrentProgramScene', {'sceneName': scene}))

    async def on_scene_item_enable_state_changed(self, event_type, event_data):
        # {'sceneItemEnabled': False, 'sceneItemId': 3, 'sceneName': 'Сцена 2'}
        if event_type != 'SceneItemEnableStateChanged':
            return
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        await self.wsS.call(simpleobsws.Request('SetSceneItemEnabled', event_data))

    async def on_scene_item_list_reindexed(self, event_type, event_data):
        # {'sceneItems': [{'sceneItemId': 4, 'sceneItemIndex': 0}, {'sceneItemId': 3, 'sceneItemIndex': 1}], 'sceneName': 'Сцена 2'}
        if event_type != 'SceneItemListReindexed':
            return
        print('Reindexing scene items.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        print()
        for item in event_data['sceneItems']:
            await self.wsS.call(simpleobsws.Request(
                'SetSceneItemIndex', {
                    'sceneName': event_data['sceneName'],
                    'sceneItemId': item['sceneItemId'],
                    'sceneItemIndex': item['sceneItemIndex']
                }
            ))

    async def on_scene_item_lock_state_changed(self, event_type, event_data):
        # {'sceneItemId': 4, 'sceneItemLocked': True, 'sceneName': 'Сцена 2'}
        if event_type != 'SceneItemLockStateChanged':
            return
        print(f'Locking scene item #{event_data["sceneItemId"]}.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        print()
        await self.wsS.call(simpleobsws.Request('SetSceneItemLocked', event_data))

    async def on_scene_item_selected(self, event_type, event_data):
        # {'sceneItemId': 2, 'sceneName': 'Сцена 2'}
        if event_type != 'SceneItemSelected':
            return
        print('Scene item selection change detected.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        print()
        await self.send_transform(force=True)

    async def on_scene_item_transform_changed(self, event_type, event_data):
        # {'sceneItemId': 4,
        #  'sceneItemTransform': {'alignment': 5, 'boundsAlignment': 0, 'boundsHeight': 0.0, 'boundsType': 'OBS_BOUNDS_NONE', 'boundsWidth': 0.0, 'cropBottom': 0, 'cropLeft': 0, 'cropRight': 0, 'cropTop': 0, 'height': 639.0000610351562, 'positionX': 781.0, 'positionY': 269.0, 'rotation': 0.0, 'scaleX': 1.065000057220459, 'scaleY': 1.065000057220459, 'sourceHeight': 600.0, 'sourceWidth': 800.0, 'width': 852.0000610351562},
        #  'sceneName': 'Сцена 2'}
        if event_type != 'SceneItemTransformChanged':
            return
        print(f'Saving transform of scene item #{event_data["sceneItemId"]}.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        print()
        if event_data['sceneItemTransform']['boundsType'] == 'OBS_BOUNDS_NONE':
            event_data['sceneItemTransform'].pop('boundsHeight')
            event_data['sceneItemTransform'].pop('boundsWidth')
        self.transform_data = event_data
        self.last_transform_update = time()

    async def send_transform(self, force=False):
        if not self.transform_data:
            return
        if time() - self.last_transform_update < 1 and not force:
            return
        print(f'Changing transform of scene item #{self.transform_data["sceneItemId"]}.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        print()
        await self.wsS.call(simpleobsws.Request('SetSceneItemTransform', self.transform_data))
        self.transform_data = None
        self.last_transform_update = time()

    async def sync_transform(self):
        while True:
            await asyncio.sleep(.1)
            await self.send_transform()

    async def on_input_created(self, event_type, event_data):
        # {'defaultInputSettings': {'css': 'body { background-color: rgba(0, 0, 0, 0); margin: 0px auto; overflow: hidden; }', 'fps': 30, 'fps_custom': False, 'height': 600, 'reroute_audio': False, 'restart_when_active': False, 'shutdown': False, 'url': '`http`s://obsproject.com/browser-source', 'webpage_control_level': 1, 'width': 800},
        #  'inputKind': 'browser_source',
        #  'inputName': 'Браузер',
        #  'inputSettings': {},
        #  'unversionedInputKind': 'browser_source'}
        if event_type != 'InputCreated':
            return
        print(f'Creating input {event_data["inputName"]}.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        print()
        await asyncio.sleep(6)
        scene = await self.wsM.call(simpleobsws.Request('GetCurrentProgramScene'))
        scene_name = scene.responseData['currentProgramSceneName']
        await self.wsS.call(simpleobsws.Request('CreateInput', {
            'sceneName': scene_name,
            'inputName': event_data['inputName'],
            'inputKind': event_data['inputKind'],
            'inputSettings': event_data['defaultInputSettings']
        }))
        await self.sync_input_settings(event_data['inputName'])

    async def on_input_removed(self, event_type, event_data):
        # {'inputName': 'Браузер'}
        if event_type != 'InputRemoved':
            return
        print(f'Removing input {event_data["inputName"]}.')
        if not self.wsM or not self.wsS:
            print('Connections with "Master" or "Slave" not established.\n')
            return
        print()
        await self.wsS.call(simpleobsws.Request('RemoveInput', event_data))

    async def sync_input_settings(self, input_name):
        req_settings = await self.wsM.call(simpleobsws.Request('GetInputSettings', {'inputName': input_name}))
        settings = req_settings.responseData['inputSettings']
        await self.wsS.call(simpleobsws.Request('SetInputSettings', {
            'inputName': input_name,
            'inputSettings': settings
        }))


if __name__ == '__main__':
    loop = asyncio.new_event_loop()
    obs = OBSsync()
    loop.run_until_complete(obs.connect_master('localhost', 4444))
    loop.run_until_complete(obs.connect_slave('localhost', 4445))
    loop.create_task(obs.sync_transform())
    loop.run_forever()
