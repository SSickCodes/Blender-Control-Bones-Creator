# Blender-Control-Bones-Creator
# This script makes it easy to deform and control face rigs in Blender.
# This is a revised script made by SSickCodes orginally created by Getgoodwithvance
# It should work well with Blender 4.2+
bl_info = {
    "name": "Control Bones Creator",
    "author": "Vince Russell (getgoodwithvince), updated for Blender 4.2 by Copilot",
    "version": (1, 3, 0),
    "blender": (4, 2, 0),
    "location": "View3D > Sidebar > Tools",
    "description": "Add controllers for bendy bones automatically",
    "category": "3D View",
}

import bpy
from mathutils import Vector

class RunAction(bpy.types.Operator):
    bl_idname = "bone.add_controller"
    bl_label = "Add Control Bones"
    bl_description = "Add control bones for selected bendy bone"
    bl_options = {'REGISTER', 'UNDO'}

    startControllerSize: bpy.props.FloatProperty(
        name="Start CTRL Size",
        default=0.1,
        description="Size of start controller",
        min=0.01
    )
    endControllerSize: bpy.props.FloatProperty(
        name="End CTRL Size",
        default=0.1,
        description="Size of end controller",
        min=0.01
    )

    def add_custom_control_empty(self, context, current_armature):
        control_obj_name = 'BENDY_CTRL'
        if control_obj_name in bpy.data.objects:
            return False
        # Switch to OBJECT mode to add empty
        bpy.ops.object.mode_set(mode='OBJECT')
        bpy.ops.object.empty_add(type='SPHERE')
        empty_obj = context.active_object
        empty_obj.name = control_obj_name
        # Hide in viewport for Blender 4.2
        empty_obj.hide_viewport = True
        # Select armature again
        context.view_layer.objects.active = current_armature
        bpy.ops.object.mode_set(mode='EDIT')
        return True

    def execute(self, context):
        obj = context.object
        if obj is None or obj.type != 'ARMATURE':
            self.report({'WARNING'}, 'Select an armature in edit mode')
            return {'CANCELLED'}

        if obj.mode != 'EDIT':
            self.report({'WARNING'}, 'Must be in edit mode')
            return {'CANCELLED'}

        selected = context.selected_editable_bones
        if len(selected) != 1:
            self.report({'WARNING'}, 'Select only 1 edit bone at a time')
            return {'CANCELLED'}

        self.current_active_bone = selected[0]
        self.current_active_bone_name = self.current_active_bone.name
        obj.data.display_type = 'BBONE'
        self.add_custom_control_empty(context, obj)
        self.create_control_bones(context, obj)
        return {'FINISHED'}

    def create_control_bones(self, context, current_armature):
        # make sure we are in edit mode
        bpy.ops.object.mode_set(mode='EDIT')
        main_bone = self.current_active_bone
        roll = main_bone.roll
        scale_x = main_bone.bbone_x
        scale_z = main_bone.bbone_z
        head = main_bone.head
        tail = main_bone.tail

        v1 = Vector((head[0] - tail[0], head[1] - tail[1], head[2] - tail[2]))
        if v1.length == 0:
            v1 = Vector((0, 1, 0))
        else:
            v1.normalize()

        name_split = main_bone.name.rsplit(sep=".")
        if len(name_split) > 1:
            start_name = f"CTRL_{name_split[0]}_Start.{name_split[1]}"
            end_name = f"CTRL_{name_split[0]}_End.{name_split[1]}"
        else:
            start_name = f"CTRL_{main_bone.name}_Start"
            end_name = f"CTRL_{main_bone.name}_End"

        # Create start bone
        start_bone = current_armature.data.edit_bones.new(name=start_name)
        start_bone.use_deform = False
        start_bone.tail = head
        start_bone.head = (
            head[0] + (v1[0] * self.startControllerSize),
            head[1] + (v1[1] * self.startControllerSize),
            head[2] + (v1[2] * self.startControllerSize)
        )
        start_bone.bbone_x = scale_x
        start_bone.bbone_z = scale_z
        start_bone.roll = roll

        # Create end bone
        end_bone = current_armature.data.edit_bones.new(name=end_name)
        end_bone.use_deform = False
        end_bone.head = tail
        end_bone.tail = (
            tail[0] + (v1[0] * -self.endControllerSize),
            tail[1] + (v1[1] * -self.endControllerSize),
            tail[2] + (v1[2] * -self.endControllerSize)
        )
        end_bone.bbone_x = scale_x
        end_bone.bbone_z = scale_z
        end_bone.roll = roll

        self.setup_ctrl_bone_relationships(context, current_armature, main_bone, start_bone, end_bone)

    def setup_ctrl_bone_relationships(self, context, current_armature, main_bone, start_bone, end_bone):
        # Parent main bone to start bone
        main_bone.parent = start_bone
        main_bone.use_connect = True

        # Set bone handle types and custom handles
        main_bone.bbone_handle_type_start = 'ABSOLUTE'
        main_bone.bbone_handle_type_end = 'ABSOLUTE'
        main_bone.bbone_custom_handle_start = start_bone
        main_bone.bbone_custom_handle_end = end_bone

        # Switch to pose mode for constraints and custom shapes
        bpy.ops.object.mode_set(mode='POSE')
        pose_bones = current_armature.pose.bones
        active_pose_bone = pose_bones[main_bone.name]
        # Add stretch_to constraint
        c = active_pose_bone.constraints.new(type="STRETCH_TO")
        c.target = current_armature
        c.subtarget = end_bone.name

        self.set_control_display_obj(context, start_bone, end_bone)
        bpy.ops.object.mode_set(mode='EDIT')

    def set_control_display_obj(self, context, start_bone, end_bone):
        ctrl_obj = bpy.data.objects.get('BENDY_CTRL')
        if ctrl_obj:
            obj = bpy.context.object
            if obj.mode != 'POSE':
                bpy.ops.object.mode_set(mode='POSE')
            pose_bones = obj.pose.bones
            if start_bone.name in pose_bones:
                pose_bones[start_bone.name].custom_shape = ctrl_obj
            if end_bone.name in pose_bones:
                pose_bones[end_bone.name].custom_shape = ctrl_obj
        return {'FINISHED'}

class ControllerPanelUI(bpy.types.Panel):
    bl_label = "CONTROL BONES CREATOR"
    bl_idname = "OBJECT_PT_control_bones_creator"
    bl_space_type = "VIEW_3D"
    bl_region_type = "UI"
    bl_category = "Tools"

    def draw(self, context):
        layout = self.layout
        obj = context.object

        row = layout.row(align=False)
        row.operator("bone.add_controller", icon='CONSTRAINT_BONE')

        if obj and obj.type == 'ARMATURE':
            if obj.mode == "EDIT":
                current_active_bone = obj.data.edit_bones.active
            elif obj.mode == "POSE":
                # In Blender 4.x, .active_bone is a Bone, not an EditBone
                current_active_bone = obj.data.bones.active if hasattr(obj.data.bones, 'active') else None
            else:
                current_active_bone = None
            if current_active_bone:
                row = layout.row(align=False)
                row.label(text="Active Bone: " + current_active_bone.name)

classes = (RunAction, ControllerPanelUI)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)

def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)

if __name__ == "__main__":
    register()
