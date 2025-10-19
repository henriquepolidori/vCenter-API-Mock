# vCenter-API-Mock


All this envoirnment It was done using Windows 11 and WSL with ubuntu 22.04.


1 - Install python 3.12 env package.

2 - Create python Envoirment variable using sintax python3.12 -m venv API-vcenter.

3 - Active python envoirment using command cd API-vcenter/bin/active

4 - install dependences: pip install fastapi "uvicorn[standard]"

5 - Save content below calling vcenter_mock_api.py


# vCenter Mock API Microservice
# Author: Gemini
# Description: A standalone FastAPI application to simulate a vCenter environment.
# This allows for testing API interactions for VM creation from templates,
# datastore selection, and network assignment without a real vCenter instance.
#
# To run this microservice:
# 1. Install the required Python libraries:
#    pip install fastapi "uvicorn[standard]"
# 2. Save this code as `vcenter_mock_api.py`.
# 3. Run the server from your terminal:
#    uvicorn vcenter_mock_api:app --reload
# 4. Access the interactive API documentation at http://127.0.0.1:8000/docs

import uuid
from fastapi import FastAPI, HTTPException, Path
from pydantic import BaseModel, Field
from typing import List, Dict

# --- Pydantic Models for our vCenter Objects ---
# These models define the structure of our API resources and request bodies.
# They also provide data validation.

class PlacementSpec(BaseModel):
    """Defines where the new VM will be placed."""
    folder: str = Field(..., description="The ID of the folder to place the VM in.")
    datastore: str = Field(..., description="The ID of the datastore for the VM's disk.")

class VmHardwareSpec(BaseModel):
    """Defines the hardware configuration for the new VM."""
    network: str = Field(..., description="The ID of the network to connect the VM to.")

class VMCloneSpec(BaseModel):
    """The request body for cloning a VM/Template."""
    name: str = Field(..., description="The name of the new virtual machine.")
    placement: PlacementSpec
    hardware: VmHardwareSpec

class VirtualMachine(BaseModel):
    """Represents a Virtual Machine or a VM Template."""
    vm_id: str = Field(default_factory=lambda: f"vm-{uuid.uuid4()}")
    name: str
    is_template: bool = False
    folder_id: str
    datastore_id: str
    network_id: str
    power_state: str = "POWERED_OFF"

class Datastore(BaseModel):
    """Represents a Datastore."""
    datastore_id: str
    name: str
    capacity_gb: int
    free_space_gb: int

class Network(BaseModel):
    """Represents a Network on a VDS."""
    network_id: str
    name: str
    vds_name: str

class Folder(BaseModel):
    """Represents a VM Folder."""
    folder_id: str
    name: str


# --- In-Memory Mock Database ---
# In a real-world scenario, this data would come from a database.
# For our mock service, we'll pre-populate some data to simulate a live environment.

mock_db = {
    "folders": {
        "folder-101": Folder(folder_id="folder-101", name="Templates"),
        "folder-102": Folder(folder_id="folder-102", name="Production VMs"),
        "folder-103": Folder(folder_id="folder-103", name="Dev VMs"),
    },
    "datastores": {
        "datastore-201": Datastore(datastore_id="datastore-201", name="DS_SSD_01", capacity_gb=1024, free_space_gb=512),
        "datastore-202": Datastore(datastore_id="datastore-202", name="DS_HDD_Archive", capacity_gb=4096, free_space_gb=2048),
    },
    "networks": {
        "network-301": Network(network_id="network-301", name="Prod_VLAN_10", vds_name="vds-production"),
        "network-302": Network(network_id="network-302", name="Dev_VLAN_20", vds_name="vds-development"),
    },
    "vms": {
        "vm-template-ubuntu20": VirtualMachine(
            vm_id="vm-template-ubuntu20",
            name="template-ubuntu-20.04",
            is_template=True,
            folder_id="folder-101",
            datastore_id="datastore-201",
            network_id="network-301" # Default network for template
        ),
         "vm-template-win2022": VirtualMachine(
            vm_id="vm-template-win2022",
            name="template-windows-2022",
            is_template=True,
            folder_id="folder-101",
            datastore_id="datastore-201",
            network_id="network-301"
        ),
    }
}


# --- FastAPI Application ---
app = FastAPI(
    title="vCenter Mock API",
    description="A simulator for vCenter Server REST APIs.",
    version="1.0.0"
)

# --- API Endpoints ---

@app.get("/api/vcenter/vm", response_model=List[VirtualMachine], tags=["Virtual Machine"])
def list_vms():
    """Retrieves a list of all virtual machines and templates."""
    return list(mock_db["vms"].values())

@app.get("/api/vcenter/datastore", response_model=List[Datastore], tags=["Datastore"])
def list_datastores():
    """Retrieves a list of all available datastores."""
    return list(mock_db["datastores"].values())

@app.get("/api/vcenter/network", response_model=List[Network], tags=["Network"])
def list_networks():
    """Retrieves a list of all available networks."""
    return list(mock_db["networks"].values())

@app.get("/api/vcenter/folder", response_model=List[Folder], tags=["Folder"])
def list_folders():
    """Retrieves a list of all available VM folders."""
    return list(mock_db["folders"].values())

@app.post("/api/vcenter/vm/{source_vm_id}/clone", response_model=VirtualMachine, status_code=201, tags=["Virtual Machine"])
def clone_vm_from_template(
    clone_spec: VMCloneSpec,
    source_vm_id: str = Path(..., description="The ID of the source VM or Template to clone.")
):
    """
    Simulates creating a new Virtual Machine by cloning from a source VM or template.
    This mimics the workflow of deploying a VM from a template.
    """
    # --- Input Validation ---
    # 1. Check if the source template/VM exists
    if source_vm_id not in mock_db["vms"]:
        raise HTTPException(status_code=404, detail=f"Source VM or Template with ID '{source_vm_id}' not found.")

    source_vm = mock_db["vms"][source_vm_id]
    if not source_vm.is_template:
        # In a real vCenter, you can clone non-templates, but for this mock we'll add a warning.
        print(f"Warning: Cloning from a VM '{source_vm.name}' that is not marked as a template.")

    # 2. Check if the target resources exist
    if clone_spec.placement.folder not in mock_db["folders"]:
        raise HTTPException(status_code=400, detail=f"Folder with ID '{clone_spec.placement.folder}' not found.")
    if clone_spec.placement.datastore not in mock_db["datastores"]:
        raise HTTPException(status_code=400, detail=f"Datastore with ID '{clone_spec.placement.datastore}' not found.")
    if clone_spec.hardware.network not in mock_db["networks"]:
        raise HTTPException(status_code=400, detail=f"Network with ID '{clone_spec.hardware.network}' not found.")

    # 3. Check for name collision in the target folder (simplified check)
    for vm in mock_db["vms"].values():
        if vm.name == clone_spec.name and vm.folder_id == clone_spec.placement.folder:
             raise HTTPException(status_code=409, detail=f"A VM with the name '{clone_spec.name}' already exists in the target folder.")


    # --- Simulate VM Creation ---
    print(f"Cloning VM from '{source_vm.name}' to new VM '{clone_spec.name}'...")
    new_vm = VirtualMachine(
        name=clone_spec.name,
        is_template=False, # Cloned VMs are not templates
        folder_id=clone_spec.placement.folder,
        datastore_id=clone_spec.placement.datastore,
        network_id=clone_spec.hardware.network,
        power_state="POWERED_OFF"
    )

    # Add the new VM to our "database"
    mock_db["vms"][new_vm.vm_id] = new_vm

    print(f"Successfully created VM '{new_vm.name}' with ID '{new_vm.vm_id}'.")
    print(f"  -> Folder: {mock_db['folders'][new_vm.folder_id].name}")
    print(f"  -> Datastore: {mock_db['datastores'][new_vm.datastore_id].name}")
    print(f"  -> Network: {mock_db['networks'][new_vm.network_id].name}")

    # Return the newly created VM object
    return new_vm

# Example of how you could add a simple root endpoint for health checks
@app.get("/", tags=["Health Check"])
def read_root():
    return {"status": "vCenter Mock API is running"}


6 - uvicorn vcenter_mock_api:app --reload

7 - Open your web browser and go to http://127.0.0.1:8000/docs
