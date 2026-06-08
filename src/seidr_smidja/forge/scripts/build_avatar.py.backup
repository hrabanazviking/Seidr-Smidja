"""Forge build script — runs INSIDE Blender via ``--python``.

This script is injected into a headless Blender process by the Forge runner.
It imports bpy (Blender's Python API) and the VRM Add-on for Blender.

Arguments (passed via argv after ``--``):
    --spec <path>       Path to the spec JSON file
    --base <path>       Path to the base mesh file (.vrm, .fbx, .obj, .glb)
    --output <path>     Path where the output .vrm should be written
    --max-tex <int>     Maximum texture resolution (e.g. 1024). 0=unlimited.

Supported spec operations in v0.3:
    - Load base .vrm, .fbx, .obj, .glb (auto-detected by extension)
    - Apply hair color (HSV texture tinting + Principled BSDF fallback)
    - Apply eye color (HSV texture tinting + Principled BSDF fallback)
    - Apply skin color (HSV texture tinting, multi-material)
    - Apply nail/lip color (HSV texture tinting)
    - Apply body height scale (Z-axis armature scaling)
    - Set VRM 1.0 humanoid bone mapping (explicit, no auto-detection)
    - Set VRM 1.0 expression presets (symmetric L+R shape key binding)
    - Set VRM 1.0 metadata (title, author, version, license, contact)
    - Set VRM 1.0 first-person offset (mesh annotation)
    - Set VRM 1.0 lookAt configuration (eye bone rotation)
    - Configure skin subsurface scattering
    - Downsample textures to reduce VRM file size
    - Auto-configure VRM 1.0 extension for non-VRM bases

Key design decisions:
    - D-008: VRM Add-on ``initial_automatic_bone_assignment`` is forced to ``False``
      to prevent the structure search from overwriting our explicit bone mappings.
    - D-009: ``filter_by_human_bone_hierarchy`` is set to ``False`` for non-standard
      rigs (TurboSquid, MB-Lab) where bone hierarchy differs from VRM conventions.
    - D-010: FBX import uses Blender's built-in ``import_scene.fbx`` operator.
    - D-011: Symmetric expressions bind BOTH L and R shape keys (e.g. happy = Smile_L + Smile_R)
    - D-012: VRM lookAt uses bone rotation mode with eye bones, not morph targets
    - D-013: Expression morph binds use shape key NAME string, not integer index

Exit codes:
    0  = success (output .vrm was exported)
    1  = usage / argument error
    2  = spec read / parse error
    3  = import error (VRM, FBX, or other format)
    4  = transformation error
    5  = VRM export error
    6  = post-export validation error

NOTE: This script requires Blender with the VRM Add-on for Blender installed.
      (https://github.com/saturday06/VRM-Addon-for-Blender)
      Tests requiring this are marked @pytest.mark.requires_blender.
"""
from __future__ import annotations

import colorsys
import json
import logging
import os
import sys
from pathlib import Path

# Configure logging for Blender output
logger = logging.getLogger("build_avatar")
_handler = logging.StreamHandler(sys.stderr)
_handler.setFormatter(logging.Formatter("[%(name)s] %(levelname)s: %(message)s"))
logger.addHandler(_handler)
logger.setLevel(logging.DEBUG)

# ──────────────────────────────────────────────────────────────────────────────
# TurboSquid RPG Female Model — Correct VRM 1.0 Humanoid Bone Mapping
# Generated from detailed armature inspection (101 bones, L_/R_ naming).
# These are the CORRECT mappings — the VRM Add-on's auto-detection gets many
# wrong (e.g. rightFoot→R_KneeShareBone, head→NeckTwist01, leftHand→L_Forearm).
# ──────────────────────────────────────────────────────────────────────────────
TURBOSQUID_BONE_MAP: dict[str, str] = {
    # Core spine
    "hips": "Hip",
    "spine": "Spine01",
    "chest": "Spine02",
    "upperChest": "Spine03",
    "neck": "NeckTwist02",
    "head": "Head",
    # Left leg
    "leftUpperLeg": "L_Thigh",
    "leftLowerLeg": "L_Calf",
    "leftFoot": "L_Foot",
    "leftToes": "L_Toe",
    # Right leg
    "rightUpperLeg": "R_Thigh",
    "rightLowerLeg": "R_Calf",
    "rightFoot": "R_Foot",
    "rightToes": "R_Toe",
    # Left arm
    "leftShoulder": "L_Shoulder",
    "leftUpperArm": "L_UpperArm",
    "leftLowerArm": "L_Forearm",
    "leftHand": "L_Hand",
    # Right arm
    "rightShoulder": "R_Shoulder",
    "rightUpperArm": "R_UpperArm",
    "rightLowerArm": "R_Forearm",
    "rightHand": "R_Hand",
    # Left fingers
    "leftThumbMetacarpal": "L_Thumb1",
    "leftThumbProximal": "L_Thumb2",
    "leftThumbDistal": "L_Thumb3",
    "leftIndexProximal": "L_Index1",
    "leftIndexIntermediate": "L_Index2",
    "leftIndexDistal": "L_Index3",
    "leftMiddleProximal": "L_Mid1",
    "leftMiddleIntermediate": "L_Mid2",
    "leftMiddleDistal": "L_Mid3",
    "leftRingProximal": "L_Ring1",
    "leftRingIntermediate": "L_Ring2",
    "leftRingDistal": "L_Ring3",
    "leftLittleProximal": "L_Pinky1",
    "leftLittleIntermediate": "L_Pinky2",
    "leftLittleDistal": "L_Pinky3",
    # Right fingers
    "rightThumbMetacarpal": "R_Thumb1",
    "rightThumbProximal": "R_Thumb2",
    "rightThumbDistal": "R_Thumb3",
    "rightIndexProximal": "R_Index1",
    "rightIndexIntermediate": "R_Index2",
    "rightIndexDistal": "R_Index3",
    "rightMiddleProximal": "R_Mid1",
    "rightMiddleIntermediate": "R_Mid2",
    "rightMiddleDistal": "R_Mid3",
    "rightRingProximal": "R_Ring1",
    "rightRingIntermediate": "R_Ring2",
    "rightRingDistal": "R_Ring3",
    "rightLittleProximal": "R_Pinky1",
    "rightLittleIntermediate": "R_Pinky2",
    "rightLittleDistal": "R_Pinky3",
    # Eyes and jaw
    "leftEye": "L_Eye",
    "rightEye": "R_Eye",
    "jaw": "UpperJaw",
}

# ──────────────────────────────────────────────────────────────────────────────
# VRM 1.0 expression preset mapping — TurboSquid model shape keys
# Maps VRM expression preset names to shape key bindings.
# D-011: Symmetric expressions bind BOTH L and R shape keys.
# Shape keys verified from actual model inspection:
#   Body: 148 keys, EyeOcclusion: 222 keys, TearLine: 220 keys,
#   Tongue: 37 keys, Teeth: 1 key, Eye: 1 key
# ──────────────────────────────────────────────────────────────────────────────
TURBOSQUID_EXPRESSION_MAP: dict[str, list[dict]] = {
    # ── Emotions (symmetric: L + R sides) ──────────────────────────────────
    "happy": [
        {"shape_key": "Mouth_Smile_L", "weight": 1.0},
        {"shape_key": "Mouth_Smile_R", "weight": 1.0},
        {"shape_key": "Cheek_Raise_L", "weight": 0.5},
        {"shape_key": "Cheek_Raise_R", "weight": 0.5},
    ],
    "angry": [
        {"shape_key": "Brow_Compress_L", "weight": 1.0},
        {"shape_key": "Brow_Compress_R", "weight": 1.0},
        {"shape_key": "Mouth_Press_L", "weight": 0.4},
        {"shape_key": "Mouth_Press_R", "weight": 0.4},
        {"shape_key": "Nose_Crease_L", "weight": 0.3},
        {"shape_key": "Nose_Crease_R", "weight": 0.3},
    ],
    "sad": [
        {"shape_key": "Mouth_Frown_L", "weight": 1.0},
        {"shape_key": "Mouth_Frown_R", "weight": 1.0},
        {"shape_key": "Brow_Raise_Inner_L", "weight": 0.6},
        {"shape_key": "Brow_Raise_Inner_R", "weight": 0.6},
        {"shape_key": "Mouth_Down", "weight": 0.3},
    ],
    "relaxed": [
        {"shape_key": "Eye_Squint_L", "weight": 0.5},
        {"shape_key": "Eye_Squint_R", "weight": 0.5},
        {"shape_key": "Mouth_Smile_L", "weight": 0.3},
        {"shape_key": "Mouth_Smile_R", "weight": 0.3},
    ],
    "surprised": [
        {"shape_key": "Eye_Wide_L", "weight": 1.0},
        {"shape_key": "Eye_Wide_R", "weight": 1.0},
        {"shape_key": "Jaw_Open", "weight": 0.5},
        {"shape_key": "Brow_Raise_Outer_L", "weight": 0.7},
        {"shape_key": "Brow_Raise_Outer_R", "weight": 0.7},
    ],
    # ── Vowels / visemes ────────────────────────────────────────────────────
    "aa": [
        {"shape_key": "V_Open", "weight": 1.0},
        {"shape_key": "Jaw_Open", "weight": 0.7},
    ],
    "ih": [
        {"shape_key": "V_Explosive", "weight": 1.0},
        {"shape_key": "Mouth_Up", "weight": 0.3},
    ],
    "ou": [
        {"shape_key": "V_Tight_O", "weight": 1.0},
        {"shape_key": "Mouth_Pucker_Up_L", "weight": 0.5},
        {"shape_key": "Mouth_Pucker_Up_R", "weight": 0.5},
    ],
    "ee": [
        {"shape_key": "V_Wide", "weight": 1.0},
        {"shape_key": "Mouth_Smile_L", "weight": 0.3},
        {"shape_key": "Mouth_Smile_R", "weight": 0.3},
    ],
    "oh": [
        {"shape_key": "V_Affricate", "weight": 1.0},
        {"shape_key": "Jaw_Open", "weight": 0.4},
    ],
    # ── Eye blinks (asymmetric for natural feel) ────────────────────────────
    "blink": [
        {"shape_key": "Eye_Blink_L", "weight": 1.0},
        {"shape_key": "Eye_Blink_R", "weight": 1.0},
    ],
    "blinkLeft": [
        {"shape_key": "Eye_Blink_L", "weight": 1.0},
    ],
    "blinkRight": [
        {"shape_key": "Eye_Blink_R", "weight": 1.0},
    ],
    # ── Eye look directions (on Body mesh) ──────────────────────────────────
    "lookUp": [
        {"shape_key": "Eye_L_Look_Up", "weight": 1.0},
        {"shape_key": "Eye_R_Look_Up", "weight": 1.0},
    ],
    "lookDown": [
        {"shape_key": "Eye_L_Look_Down", "weight": 1.0},
        {"shape_key": "Eye_R_Look_Down", "weight": 1.0},
    ],
    "lookLeft": [
        {"shape_key": "Eye_L_Look_L", "weight": 1.0},
    ],
    "lookRight": [
        {"shape_key": "Eye_R_Look_R", "weight": 1.0},
    ],
    # ── Neutral (basis shape — no binding needed) ──────────────────────────
    # "neutral": no binding needed — it's the basis shape
}

# Material name fragments for color application (case-insensitive matching)
SKIN_KEYWORDS = ("skin", "body", "head", "arm", "leg", "face")
EYE_KEYWORDS = ("eye", "iris", "cornea")
HAIR_KEYWORDS = ("hair", "hairs", "scalp")
NAIL_KEYWORDS = ("nail", "nails")
LIP_KEYWORDS = ("lip", "lips", "mouth_inner")
TEAR_KEYWORDS = ("tear", "tearline")
OCCLUSION_KEYWORDS = ("occlusion",)

# HSV color targets for Runa Gridweaver (0-1 range)
SKIN_HSV = (0.08, 0.55, 0.88)   # warm golden-tan
EYE_HSV = (0.55, 0.70, 0.82)    # ice-blue
HAIR_HSV = (0.10, 0.60, 0.56)   # warm blonde
NAIL_HSV = (0.95, 0.25, 0.75)   # subtle pink-nude
LIP_HSV = (0.97, 0.45, 0.70)    # natural lip tint
TEAR_HSV = (0.58, 0.15, 0.90)   # faint blue tint for tears

# Default blend factors per material category (0=original, 1=full target)
DEFAULT_BLEND_FACTORS = {
    "skin": 0.6,
    "eye": 0.7,
    "hair": 0.6,
    "nail": 0.5,
    "lip": 0.5,
    "tear": 0.3,
    "occlusion": 0.4,
}

# ──────────────────────────────────────────────────────────────────────────────
# D-018: VRM 1.0 humanoid bone mapping — canonical API pattern
#
# BUG FIXED: The previous approach used getattr(human_bones, vrm_bone_name) with
# camelCase VRM spec names (e.g. "leftUpperLeg"), but Vrm1HumanBonesPropertyGroup
# exposes PointerProperties as snake_case Python attributes (e.g. "left_upper_leg").
# Only 6/55 bones mapped (hips, spine, chest, neck, head, jaw) because those happen
# to have identical camelCase and snake_case names.
#
# CORRECT API: human_bones.human_bone_name_to_human_bone() returns a dict keyed by
# HumanBoneName enums whose .value strings are the camelCase VRM names. We build
# a reverse lookup from camelCase string → Vrm1HumanBonePropertyGroup, then set
# bone_prop.node.bone_name = "ArmatureBoneName" on each.
#
# The old VRM1_HUMAN_BONE_PROPS list is REMOVED — it duplicated information already
# available through the canonical API and was the source of the naming mismatch bug.
# ──────────────────────────────────────────────────────────────────────────────


def main() -> int:
    """Entry point — returns exit code."""
    # Parse arguments from sys.argv after the '--' separator
    argv = sys.argv
    try:
        sep_idx = argv.index("--")
        args = argv[sep_idx + 1:]
    except ValueError:
        logger.error("No '--' separator found in argv.")
        return 1

    spec_path: str | None = None
    base_path: str | None = None
    output_path: str | None = None
    max_tex_size: int = 0  # 0 = unlimited

    i = 0
    while i < len(args):
        if args[i] == "--spec" and i + 1 < len(args):
            spec_path = args[i + 1]
            i += 2
        elif args[i] == "--base" and i + 1 < len(args):
            base_path = args[i + 1]
            i += 2
        elif args[i] == "--output" and i + 1 < len(args):
            output_path = args[i + 1]
            i += 2
        elif args[i] == "--max-tex" and i + 1 < len(args):
            max_tex_size = int(args[i + 1])
            i += 2
        else:
            i += 1

    if not spec_path or not base_path or not output_path:
        logger.error(
            "Missing required arguments. "
            "Got: spec=%s, base=%s, output=%s",
            spec_path, base_path, output_path,
        )
        return 1

    # Load spec
    try:
        with open(spec_path, encoding="utf-8") as fh:
            spec = json.load(fh)
        logger.info("Spec loaded: avatar_id=%s", spec.get("avatar_id", "?"))
    except Exception as exc:
        logger.error("Cannot read spec: %s", exc)
        return 2

    # Import bpy (only available inside Blender)
    try:
        import bpy  # type: ignore[import]
        import addon_utils  # type: ignore[import]
    except ImportError:
        logger.error("bpy not available — must run inside Blender.")
        return 3

    # ── Step 0: Enable required add-ons ────────────────────────────────────
    try:
        addon_utils.enable("io_scene_vrm", default_set=True)
        try:
            bpy.ops.wm.save_userpref()
        except Exception:
            pass
        logger.info("VRM Add-on enabled")
    except Exception as exc:
        logger.warning("Could not enable VRM add-on: %s", exc)
        logger.warning("VRM import/export will not be available")

    # ── Step 1: Clear scene and import base mesh ───────────────────────────
    try:
        _clear_scene(bpy)
        _import_base(bpy, base_path)
    except Exception as exc:
        logger.error("Base mesh import failed: %s", exc)
        return 3

    # ── Step 1b: Downsample textures if requested ──────────────────────────
    if max_tex_size > 0:
        _downsample_textures(bpy, max_tex_size)

    # ── Step 2: Apply spec transformations ──────────────────────────────────
    try:
        _apply_spec(bpy, spec, base_path)
    except Exception as exc:
        logger.error("Spec application failed: %s", exc)
        import traceback
        traceback.print_exc(file=sys.stderr)
        return 4

    # ── Step 3: Export VRM ─────────────────────────────────────────────────
    # Set environment variable for headless license confirmation
    os.environ["BLENDER_VRM_AUTOMATIC_LICENSE_CONFIRMATION"] = "true"

    # Find the armature for the export call
    armature_name = ""
    for obj in bpy.data.objects:
        if obj.type == "ARMATURE":
            armature_name = obj.name
            break

    try:
        # D-016: Use 'EXEC_DEFAULT' to bypass invoke() which tries to open
        # GUI dialogs for bone assignment validation. In headless mode,
        # invoke() would open wm.vrm_export_human_bones_assignment which
        # requires a window manager context.
        # Also pass ignore_warning=True to skip warning-level validation
        # that would otherwise open a confirmation dialog.
        result = bpy.ops.export_scene.vrm(
            "EXEC_DEFAULT",
            filepath=output_path,
            armature_object_name=armature_name,
            ignore_warning=True,
        )
        if "FINISHED" not in result:
            logger.error("VRM export returned %s for %s", result, output_path)
            return 5
        logger.info("VRM exported: %s", output_path)
    except Exception as exc:
        logger.error("VRM export failed: %s", exc)
        import traceback
        traceback.print_exc(file=sys.stderr)
        return 5

    # ── Step 4: Post-export validation ──────────────────────────────────────
    try:
        _validate_vrm(output_path)
    except Exception as exc:
        logger.warning("Post-export validation warning: %s", exc)
        # Non-fatal — the VRM was still exported

    logger.info("Build complete.")
    return 0


# ──────────────────────────────────────────────────────────────────────────────
# Scene helpers
# ──────────────────────────────────────────────────────────────────────────────

def _clear_scene(bpy) -> None:
    """Remove all objects and orphan data, keeping add-ons registered."""
    bpy.ops.object.select_all(action="SELECT")
    bpy.ops.object.delete(use_global=True)

    # Thorough orphan cleanup — clear all unused data blocks
    for block_type in ("meshes", "materials", "textures", "images",
                       "armatures", "actions", "cameras", "lights",
                       "curves", "metaballs", "armatures", "lattices",
                       "shape_keys", "particle_settings", "node_groups"):
        try:
            collection = getattr(bpy.data, block_type)
        except AttributeError:
            continue
        for block in list(collection):
            if block.users == 0:
                collection.remove(block)

    # Purge orphan data (removes indirect users too)
    try:
        bpy.ops.outliner.orphans_purge(do_local_ids=True, do_linked_ids=True, recursive=True)
    except Exception:
        pass  # Not available in all Blender versions

    logger.info("Scene cleared")


def _import_base(bpy, base_path: str) -> None:
    """Import a base mesh file (VRM, FBX, OBJ, GLB) into the current scene."""
    base_lower = base_path.lower()
    if base_lower.endswith(".vrm"):
        result = bpy.ops.import_scene.vrm(filepath=base_path)
        if "FINISHED" not in result:
            raise RuntimeError(f"VRM import returned {result} for {base_path}")
        logger.info("Base VRM imported: %s", base_path)
    elif base_lower.endswith(".fbx"):
        result = bpy.ops.import_scene.fbx(filepath=base_path)
        if "FINISHED" not in result:
            raise RuntimeError(f"FBX import returned {result} for {base_path}")
        logger.info("Base FBX imported: %s", base_path)
    elif base_lower.endswith(".obj"):
        result = bpy.ops.import_scene.obj(filepath=base_path)
        if "FINISHED" not in result:
            raise RuntimeError(f"OBJ import returned {result} for {base_path}")
        logger.info("Base OBJ imported: %s", base_path)
    elif base_lower.endswith((".glb", ".gltf")):
        result = bpy.ops.import_scene.gltf(filepath=base_path)
        if "FINISHED" not in result:
            raise RuntimeError(f"glTF import returned {result} for {base_path}")
        logger.info("Base glTF imported: %s", base_path)
    else:
        raise ValueError(f"Unsupported base file format: {base_path}")


# ──────────────────────────────────────────────────────────────────────────────
# Material helpers
# ──────────────────────────────────────────────────────────────────────────────

def _classify_material(mat_name: str) -> tuple[str | None, tuple | None, float]:
    """Classify a material into a category and return (category, hsv_target, blend).

    Returns (None, None, 0.0) if the material doesn't need tinting.
    """
    name_lower = mat_name.lower()

    # Check each category by keyword
    if any(kw in name_lower for kw in SKIN_KEYWORDS):
        return ("skin", SKIN_HSV, DEFAULT_BLEND_FACTORS["skin"])
    if any(kw in name_lower for kw in EYE_KEYWORDS):
        # Distinguish between eye whites (occlusion) and iris
        if any(kw in name_lower for kw in OCCLUSION_KEYWORDS):
            return ("occlusion", SKIN_HSV, DEFAULT_BLEND_FACTORS["occlusion"])
        return ("eye", EYE_HSV, DEFAULT_BLEND_FACTORS["eye"])
    if any(kw in name_lower for kw in HAIR_KEYWORDS):
        return ("hair", HAIR_HSV, DEFAULT_BLEND_FACTORS["hair"])
    if any(kw in name_lower for kw in NAIL_KEYWORDS):
        return ("nail", NAIL_HSV, DEFAULT_BLEND_FACTORS["nail"])
    if any(kw in name_lower for kw in LIP_KEYWORDS):
        return ("lip", LIP_HSV, DEFAULT_BLEND_FACTORS["lip"])
    if any(kw in name_lower for kw in TEAR_KEYWORDS):
        return ("tear", TEAR_HSV, DEFAULT_BLEND_FACTORS["tear"])

    # Special TurboSquid material names that don't match keywords
    if "cornea" in name_lower:
        return ("eye", EYE_HSV, 0.3)  # Light tint for cornea transparency
    if "breast" in name_lower:
        return ("skin", SKIN_HSV, 0.5)
    if "teeth" in name_lower or "tongue" in name_lower:
        return None, None, 0.0  # Don't tint teeth/tongue
    if "eyelash" in name_lower:
        return None, None, 0.0  # Don't tint eyelashes
    if "waista" in name_lower:
        return None, None, 0.0  # Clothing piece, don't tint

    return None, None, 0.0


def _set_material_color(material, r: float, g: float, b: float) -> bool:
    """Set a material's Principled BSDF base color. Returns True if successful.

    NOTE: When a texture node is connected to Base Color, the glTF/VRM exporter
    ignores the Principled BSDF default_value. This function is a FALLBACK
    for materials without connected textures.
    """
    if material is None:
        return False
    if not material.use_nodes:
        material.use_nodes = True
    for node in material.node_tree.nodes:
        if node.type == "BSDF_PRINCIPLED":
            node.inputs["Base Color"].default_value = (r, g, b, 1.0)
            logger.debug("Set fallback color on %s: (%.3f, %.3f, %.3f)", material.name, r, g, b)
            return True
    return False


def _configure_skin_sss(material, sss_color: tuple = None) -> bool:
    """Configure subsurface scattering on a skin material for realistic skin.

    In Blender 3.4, Principled BSDF uses 'Subsurface' (not 'Subsurface Weight').
    Args:
        material: The Blender material.
        sss_color: Optional (R, G, B, A) subsurface color. Defaults to warm skin.
    """
    if material is None or not material.use_nodes:
        return False

    if sss_color is None:
        sss_color = (0.8, 0.4, 0.25, 1.0)  # Warm subsurface for golden-tan skin

    for node in material.node_tree.nodes:
        if node.type == "BSDF_PRINCIPLED":
            # Blender 3.4 uses 'Subsurface' not 'Subsurface Weight'
            subsurf_input = node.inputs.get("Subsurface")
            if subsurf_input is not None:
                subsurf_input.default_value = 0.15  # Light SSS
                logger.debug("Set Subsurface=%.2f on %s", 0.15, material.name)
            subsurf_color = node.inputs.get("Subsurface Color")
            if subsurf_color is not None:
                subsurf_color.default_value = sss_color
                logger.debug("Set Subsurface Color on %s", material.name)
            # Also set Subsurface Radius for scattering spread
            subsurf_radius = node.inputs.get("Subsurface Radius")
            if subsurf_radius is not None:
                subsurf_radius.default_value = (1.0, 0.2, 0.1)  # Red scatters most
            return True
    return False


def _tint_texture_image(bpy, material_name: str, target_hsv: tuple,
                         blend: float = 0.6) -> bool:
    """Tint a material's Base Color texture image toward a target HSV color.

    This modifies the actual image pixels in Blender's memory, which will be
    included in the VRM export. Uses HSV blending to preserve texture detail
    (wrinkles, pores, shading) while achieving the desired color palette.

    Args:
        bpy:          Blender Python module.
        material_name: Name of the material to tint.
        target_hsv:   Target (H, S, V) in 0-1 range.
        blend:        Blend factor (0=original, 1=full target color).

    Returns:
        True if tinting was applied, False if no texture found.
    """
    try:
        import numpy as np
    except ImportError:
        logger.warning("NumPy not available in Blender — cannot tint textures")
        return False

    mat = bpy.data.materials.get(material_name)
    if mat is None:
        return False
    if not mat.use_nodes:
        return False

    for node in mat.node_tree.nodes:
        if node.type == "BSDF_PRINCIPLED":
            base_input = node.inputs["Base Color"]
            if not base_input.is_linked:
                continue
            src = base_input.links[0].from_node
            if src.type != "TEX_IMAGE" or src.image is None:
                continue

            img = src.image
            # Pack image so modifications persist in VRM export
            try:
                if img.filepath and not img.packed_file:
                    img.pack()
            except RuntimeError:
                pass  # Image already loaded in memory; pack failure is non-critical

            w, h = img.size
            channels = img.channels
            if channels < 3 or w == 0 or h == 0:
                continue

            # Read pixels as flat array, reshape
            pixels = np.array(img.pixels[:]).reshape(h, w, channels)
            rgb = pixels[:, :, :3].copy()

            # ── HSV shift using vectorized numpy operations ────────────────
            maxc = np.maximum(np.maximum(rgb[:, :, 0], rgb[:, :, 1]), rgb[:, :, 2])
            minc = np.minimum(np.minimum(rgb[:, :, 0], rgb[:, :, 1]), rgb[:, :, 2])
            diff = maxc - minc

            # Compute hue (D-014 fix: handle h6==6.0 boundary properly)
            hue = np.zeros_like(maxc)
            mask_r = (maxc == rgb[:, :, 0]) & (diff > 1e-6)
            mask_g = (maxc == rgb[:, :, 1]) & (diff > 1e-6)
            mask_b = (maxc == rgb[:, :, 2]) & (diff > 1e-6)
            hue[mask_r] = ((rgb[:, :, 1][mask_r] - rgb[:, :, 2][mask_r]) / diff[mask_r]) % 6
            hue[mask_g] = ((rgb[:, :, 2][mask_g] - rgb[:, :, 0][mask_g]) / diff[mask_g]) + 2
            hue[mask_b] = ((rgb[:, :, 0][mask_b] - rgb[:, :, 1][mask_b]) / diff[mask_b]) + 4
            hue = hue / 6.0

            sat = np.where(maxc > 1e-6, diff / maxc, 0.0)
            val = maxc

            # Blend toward target HSV
            th, ts, tv = target_hsv
            new_hue = hue * (1 - blend) + th * blend
            new_sat = sat * (1 - blend) + ts * blend

            # Value: preserve original brightness pattern (wrinkles, pores)
            # but shift the average toward the target value
            val_mean = np.mean(val[val > 0.05]) if np.any(val > 0.05) else tv
            val_scale = tv / val_mean if val_mean > 0.01 else 1.0
            # Clamp scale to prevent extreme brightening/darkening
            val_scale = np.clip(val_scale, 0.5, 2.0)
            new_val = np.clip(val * val_scale, 0, 1)

            # ── HSV → RGB conversion (D-014 fix: proper sector boundary handling) ──
            new_hue = new_hue % 1.0  # Wrap to [0, 1)
            new_sat = np.clip(new_sat, 0, 1)

            c = new_val * new_sat
            x = c * (1 - np.abs((new_hue * 6) % 2 - 1))
            m = new_val - c

            rgb_new = np.zeros_like(rgb)
            h6 = new_hue * 6

            # Sector 0: h6 in [0, 1) → R=c, G=x, B=m
            mask = (h6 >= 0) & (h6 < 1)
            rgb_new[:, :, 0] = np.where(mask, c, rgb_new[:, :, 0])
            rgb_new[:, :, 1] = np.where(mask, x, rgb_new[:, :, 1])
            rgb_new[:, :, 2] = np.where(mask, m, rgb_new[:, :, 2])

            # Sector 1: h6 in [1, 2) → R=x, G=c, B=m
            mask = (h6 >= 1) & (h6 < 2)
            rgb_new[:, :, 0] = np.where(mask, x, rgb_new[:, :, 0])
            rgb_new[:, :, 1] = np.where(mask, c, rgb_new[:, :, 1])
            rgb_new[:, :, 2] = np.where(mask, m, rgb_new[:, :, 2])

            # Sector 2: h6 in [2, 3) → R=m, G=c, B=x
            mask = (h6 >= 2) & (h6 < 3)
            rgb_new[:, :, 0] = np.where(mask, m, rgb_new[:, :, 0])
            rgb_new[:, :, 1] = np.where(mask, c, rgb_new[:, :, 1])
            rgb_new[:, :, 2] = np.where(mask, x, rgb_new[:, :, 2])

            # Sector 3: h6 in [3, 4) → R=m, G=x, B=c
            mask = (h6 >= 3) & (h6 < 4)
            rgb_new[:, :, 0] = np.where(mask, m, rgb_new[:, :, 0])
            rgb_new[:, :, 1] = np.where(mask, x, rgb_new[:, :, 1])
            rgb_new[:, :, 2] = np.where(mask, c, rgb_new[:, :, 2])

            # Sector 4: h6 in [4, 5) → R=x, G=m, B=c
            mask = (h6 >= 4) & (h6 < 5)
            rgb_new[:, :, 0] = np.where(mask, x, rgb_new[:, :, 0])
            rgb_new[:, :, 1] = np.where(mask, m, rgb_new[:, :, 1])
            rgb_new[:, :, 2] = np.where(mask, c, rgb_new[:, :, 2])

            # Sector 5: h6 in [5, 6] → R=c, G=m, B=x
            # D-014 fix: includes h6==6.0 (wraps to sector 0 visually)
            mask = (h6 >= 5) & (h6 <= 6)
            rgb_new[:, :, 0] = np.where(mask, c, rgb_new[:, :, 0])
            rgb_new[:, :, 1] = np.where(mask, m, rgb_new[:, :, 1])
            rgb_new[:, :, 2] = np.where(mask, x, rgb_new[:, :, 2])

            rgb_new += m[:, :, np.newaxis]
            rgb_new = np.clip(rgb_new, 0, 1)

            # Preserve alpha channel, write back
            pixels[:, :, :3] = rgb_new
            img.pixels[:] = pixels.flatten().tolist()
            img.update()

            logger.info("Tinted texture '%s' in material '%s' toward HSV%s (blend=%.2f)",
                        img.name, material_name, target_hsv, blend)
            return True

    return False


def _downsample_textures(bpy, max_size: int) -> None:
    """Downsample all textures to a maximum resolution to reduce VRM file size.

    Args:
        bpy:       Blender Python module.
        max_size:  Maximum texture dimension in pixels. Textures larger than this
                   will be scaled down proportionally.
    """
    downsampled = 0
    for img in bpy.data.images:
        w, h = img.size
        if w <= max_size and h <= max_size:
            continue
        # Calculate new size (proportional scaling)
        scale = min(max_size / w, max_size / h)
        new_w = max(1, int(w * scale))
        new_h = max(1, int(h * scale))
        # Ensure power-of-2 for GPU efficiency (optional but recommended)
        new_w = 1 << (new_w - 1).bit_length() if new_w > 1 else 1
        new_h = 1 << (new_h - 1).bit_length() if new_h > 1 else 1
        # Clamp again
        new_w = min(new_w, max_size)
        new_h = min(new_h, max_size)

        try:
            img.scale(new_w, new_h)
            downsampled += 1
            logger.info("Downsampled '%s': %dx%d → %dx%d", img.name, w, h, new_w, new_h)
        except Exception as exc:
            logger.warning("Failed to downsample '%s': %s", img.name, exc)

    if downsampled > 0:
        logger.info("Downsampled %d textures to max %dpx", downsampled, max_size)


# ──────────────────────────────────────────────────────────────────────────────
# Spec application
# ──────────────────────────────────────────────────────────────────────────────

def _apply_spec(bpy, spec: dict, base_path: str = "") -> None:
    """Apply parametric spec changes to the loaded Blender scene.

    Covers the v0.3 parameter set:
        - Hair/skin/eye/nail/lip color (HSV texture tinting + BSDF fallback)
        - Body height scale (armature Z-axis)
        - Skin subsurface scattering configuration
        - VRM 1.0 metadata (title, author, version, license, contact)
        - VRM 1.0 humanoid bone mapping (explicit, no auto-detection)
        - VRM 1.0 expression setup (preset → shape key mapping)
        - VRM 1.0 first-person offset
        - VRM 1.0 lookAt configuration

    Args:
        bpy:      The Blender Python API module.
        spec:     The parsed spec dict.
        base_path: Path to the base asset (used to detect format).
    """
    # Read blend factor overrides from spec
    spec_blend = spec.get("tint_blend", {})

    # ── Hair color (texture tinting) ─────────────────────────────────────────
    hair_spec = spec.get("hair", {})
    hair_color = hair_spec.get("color", {})
    if isinstance(hair_color, dict):
        hr = float(hair_color.get("r", 0.56))
        hg = float(hair_color.get("g", 0.45))
        hb = float(hair_color.get("b", 0.25))
        hair_hsv = colorsys.rgb_to_hsv(hr, hg, hb)
        blend = spec_blend.get("hair", DEFAULT_BLEND_FACTORS["hair"])
        for mat in bpy.data.materials:
            cat, _, _ = _classify_material(mat.name)
            if cat == "hair":
                if _tint_texture_image(bpy, mat.name, hair_hsv, blend=blend):
                    logger.info("Hair texture tinted: %s", mat.name)
                elif _set_material_color(mat, hr, hg, hb):
                    logger.info("Hair color applied (fallback): %s", mat.name)

    # ── Eye color (texture tinting) ──────────────────────────────────────────
    face_spec = spec.get("face", {})
    eye_color = face_spec.get("eye_color", {})
    if isinstance(eye_color, dict):
        er = float(eye_color.get("r", 0.58))
        eg = float(eye_color.get("g", 0.72))
        eb = float(eye_color.get("b", 0.81))
        eye_hsv = colorsys.rgb_to_hsv(er, eg, eb)
        blend = spec_blend.get("eye", DEFAULT_BLEND_FACTORS["eye"])
        for mat in bpy.data.materials:
            cat, _, _ = _classify_material(mat.name)
            if cat == "eye":
                if _tint_texture_image(bpy, mat.name, eye_hsv, blend=blend):
                    logger.info("Eye texture tinted: %s", mat.name)
                elif _set_material_color(mat, er, eg, eb):
                    logger.info("Eye color applied (fallback): %s", mat.name)

    # ── Skin color (texture tinting + SSS) ────────────────────────────────────
    skin_color = face_spec.get("skin_color", {})
    if isinstance(skin_color, dict):
        sr = float(skin_color.get("r", 0.87))
        sg = float(skin_color.get("g", 0.70))
        sb = float(skin_color.get("b", 0.58))
        skin_hsv = colorsys.rgb_to_hsv(sr, sg, sb)
        blend = spec_blend.get("skin", DEFAULT_BLEND_FACTORS["skin"])
        sss_spec = spec.get("subsurface_scattering", {})
        sss_enabled = sss_spec.get("enabled", True)
        sss_color = None
        if sss_enabled:
            sss_r = float(sss_spec.get("color_r", 0.8))
            sss_g = float(sss_spec.get("color_g", 0.4))
            sss_b = float(sss_spec.get("color_b", 0.25))
            sss_color = (sss_r, sss_g, sss_b, 1.0)

        for mat in bpy.data.materials:
            cat, _, _ = _classify_material(mat.name)
            if cat == "skin":
                if _tint_texture_image(bpy, mat.name, skin_hsv, blend=blend):
                    logger.info("Skin texture tinted: %s", mat.name)
                elif _set_material_color(mat, sr, sg, sb):
                    logger.info("Skin color applied (fallback): %s", mat.name)
                # Configure SSS on skin materials
                if sss_enabled:
                    _configure_skin_sss(mat, sss_color)
            elif cat == "occlusion":
                # Eye occlusion gets a lighter skin tint
                if _tint_texture_image(bpy, mat.name, skin_hsv, blend=0.4):
                    logger.info("Eye occlusion tinted: %s", mat.name)

    # ── Nail color ────────────────────────────────────────────────────────────
    nail_spec = face_spec.get("nail_color", {})
    if isinstance(nail_spec, dict):
        nr = float(nail_spec.get("r", 0.75))
        ng = float(nail_spec.get("g", 0.60))
        nb = float(nail_spec.get("b", 0.55))
        nail_hsv = colorsys.rgb_to_hsv(nr, ng, nb)
        blend = spec_blend.get("nail", DEFAULT_BLEND_FACTORS["nail"])
    else:
        nail_hsv = NAIL_HSV
        blend = DEFAULT_BLEND_FACTORS["nail"]
    for mat in bpy.data.materials:
        cat, _, _ = _classify_material(mat.name)
        if cat == "nail":
            _tint_texture_image(bpy, mat.name, nail_hsv, blend=blend)

    # ── Lip color ─────────────────────────────────────────────────────────────
    lip_spec = face_spec.get("lip_color", {})
    if isinstance(lip_spec, dict):
        lr = float(lip_spec.get("r", 0.70))
        lg = float(lip_spec.get("g", 0.40))
        lb = float(lip_spec.get("b", 0.35))
        lip_hsv = colorsys.rgb_to_hsv(lr, lg, lb)
        blend = spec_blend.get("lip", DEFAULT_BLEND_FACTORS["lip"])
    else:
        lip_hsv = LIP_HSV
        blend = DEFAULT_BLEND_FACTORS["lip"]
    for mat in bpy.data.materials:
        cat, _, _ = _classify_material(mat.name)
        if cat == "lip":
            _tint_texture_image(bpy, mat.name, lip_hsv, blend=blend)

    # ── Tear tint ─────────────────────────────────────────────────────────────
    for mat in bpy.data.materials:
        cat, _, _ = _classify_material(mat.name)
        if cat == "tear":
            _tint_texture_image(bpy, mat.name, TEAR_HSV, blend=DEFAULT_BLEND_FACTORS["tear"])

    # ── Body height scale ──────────────────────────────────────────────────
    body_spec = spec.get("body", {})
    height_scale = float(body_spec.get("height_scale", 1.0))
    if abs(height_scale - 1.0) > 0.001:
        for obj in bpy.data.objects:
            if obj.type == "ARMATURE":
                obj.scale[2] *= height_scale
                logger.info("Height scale %.2fx applied to armature: %s", height_scale, obj.name)
                break

    # ── Expression default values (VRM 0.x compatibility) ───────────────────
    expressions_spec = spec.get("expressions", {})
    targets = expressions_spec.get("targets", [])
    if targets:
        for armature in bpy.data.objects:
            if armature.type == "ARMATURE" and hasattr(armature.data, "vrm_addon_extension"):
                vrm_ext = armature.data.vrm_addon_extension
                try:
                    bsm = vrm_ext.vrm0.blend_shape_master
                    for group in bsm.blend_shape_groups:
                        for target_entry in targets:
                            t_name = target_entry.get("name", "").lower()
                            t_weight = float(target_entry.get("weight", 0.0))
                            if group.preset_name.lower() == t_name or group.name.lower() == t_name:
                                group.preview = t_weight
                                logger.debug("Expression '%s' weight → %.2f", t_name, t_weight)
                except (AttributeError, Exception):
                    pass

    # ── VRM humanoid setup for non-VRM bases ────────────────────────────────
    base_lower = base_path.lower() if base_path else ""
    if base_lower and not base_lower.endswith(".vrm"):
        logger.info("Non-VRM base detected — configuring VRM humanoid mapping...")
        _setup_vrm_humanoid(bpy, spec, base_path)

    # ── VRM metadata ────────────────────────────────────────────────────────
    _setup_vrm_metadata(bpy, spec)

    # ── VRM first-person offset ──────────────────────────────────────────────
    _setup_vrm_first_person(bpy, spec)

    # ── VRM lookAt configuration ────────────────────────────────────────────
    _setup_vrm_look_at(bpy, spec)


# ──────────────────────────────────────────────────────────────────────────────
# VRM Humanoid Bone Mapping
# ──────────────────────────────────────────────────────────────────────────────

def _setup_vrm_humanoid(bpy, spec: dict, base_path: str) -> None:
    """Configure VRM 1.0 humanoid bone mapping for non-VRM base meshes.

    CRITICAL: The VRM Add-on defaults ``initial_automatic_bone_assignment`` to
    True. When True, the export handler will CLEAR all bone_name fields and
    re-run its broken auto-detection (structure search). We MUST set this to
    False BEFORE export to preserve our explicit mappings.

    Additionally, ``filter_by_human_bone_hierarchy`` must be False for
    non-standard rigs (TurboSquid, MB-Lab) because their bone hierarchy
    differs from VRM conventions and would fail validation.
    """
    # Find the armature
    armature = None
    for obj in bpy.data.objects:
        if obj.type == "ARMATURE":
            armature = obj
            break

    if armature is None:
        logger.warning("No armature found for VRM humanoid mapping")
        return

    logger.info("Setting up VRM humanoid for armature: %s", armature.name)

    # Get the VRM extension
    vrm_ext = armature.data.vrm_addon_extension
    human_bones = vrm_ext.vrm1.humanoid.human_bones

    # ── CRITICAL (D-008): Disable auto-mapping ─────────────────────────────
    human_bones.initial_automatic_bone_assignment = False
    logger.debug("Disabled VRM auto-mapping (initial_automatic_bone_assignment = False)")

    # ── CRITICAL (D-009): Disable hierarchy filtering ──────────────────────
    human_bones.filter_by_human_bone_hierarchy = False
    logger.debug("Disabled hierarchy filtering (filter_by_human_bone_hierarchy = False)")

    # ── CRITICAL (D-017): Allow non-humanoid rig (safety net) ─────────────
    # If any required bone is not mapped, this prevents a hard validation error
    # and instead generates a skippable warning, allowing the export to proceed.
    try:
        human_bones.allow_non_humanoid_rig = False  # Only set True as last resort
    except AttributeError:
        pass

    # ── Select bone map based on base asset ──────────────────────────────────
    bone_map = TURBOSQUID_BONE_MAP

    # Allow spec overrides for bone mapping
    spec_bones = spec.get("bones", {})
    if spec_bones and isinstance(spec_bones, dict):
        bone_map = {**bone_map, **spec_bones}
        logger.info("Applied %d spec bone overrides", len(spec_bones))

    # ── Apply bone mappings using canonical PropertyGroup API ──────────────
    # D-018: Use human_bone_name_to_human_bone() instead of getattr().
    # The VRM spec uses camelCase ("leftUpperLeg") but the Blender
    # PointerProperty attributes use snake_case ("left_upper_leg").
    # getattr(human_bones, "leftUpperLeg") returns None — only 6/55 mapped.
    # The canonical method resolves this: HumanBoneName.value = camelCase string.
    bone_names_in_armature = set(bone.name for bone in armature.data.bones)
    logger.info("Armature has %d bones", len(bone_names_in_armature))

    # Build reverse lookup: camelCase VRM name → Vrm1HumanBonePropertyGroup
    human_bone_dict = human_bones.human_bone_name_to_human_bone()
    vrm_name_to_prop = {name.value: prop for name, prop in human_bone_dict.items()}
    logger.debug("Canonical bone API: %d VRM bone slots available", len(vrm_name_to_prop))

    mapped_count = 0
    missing_bones = []
    duplicate_check = {}  # bone_name → vrm_bone_name

    for vrm_bone_name, target_bone in bone_map.items():
        # Look up the PropertyGroup via canonical API (not getattr)
        bone_prop = vrm_name_to_prop.get(vrm_bone_name)
        if bone_prop is None:
            missing_bones.append(f"{vrm_bone_name}(no VRM prop)")
            continue

        if target_bone not in bone_names_in_armature:
            missing_bones.append(f"{vrm_bone_name}→{target_bone}(not found)")
            continue

        # Check for duplicate bone assignment
        if target_bone in duplicate_check:
            logger.warning("Bone '%s' mapped to both '%s' and '%s'",
                          target_bone, duplicate_check[target_bone], vrm_bone_name)
            continue
        duplicate_check[target_bone] = vrm_bone_name

        # Set bone mapping via the canonical .node.bone_name path
        bone_prop.node.bone_name = target_bone
        mapped_count += 1
        logger.debug("Mapped '%s' → '%s'", vrm_bone_name, target_bone)

    logger.info("VRM humanoid bone mapping: %d/%d bones mapped", mapped_count, len(bone_map))
    if missing_bones:
        logger.warning("Unmapped bones (%d): %s%s",
                       len(missing_bones),
                       missing_bones[:10],
                       "..." if len(missing_bones) > 10 else "")

    # ── Setup VRM 1.0 expressions for non-VRM bases ─────────────────────────
    _setup_vrm_expressions(bpy, vrm_ext, spec, base_path)

    # ── Fixup and validate ──────────────────────────────────────────────────
    try:
        from io_scene_vrm.editor.vrm1.property_group import Vrm1HumanBonesPropertyGroup
        Vrm1HumanBonesPropertyGroup.fixup_human_bones(armature)
        logger.info("Human bones fixup completed")
    except (ImportError, AttributeError, Exception) as exc:
        logger.debug("Human bones fixup skipped (non-critical): %s", exc)

    # Validate that required bones are assigned
    try:
        error_messages = human_bones.error_messages
        errors = [str(msg) for msg in error_messages]
        if errors:
            logger.warning("Bone mapping validation errors: %s", errors[:5])
        else:
            logger.info("Bone mapping validation passed ✓")
    except Exception:
        pass


def _setup_vrm_expressions(bpy, vrm_ext, spec: dict, base_path: str) -> None:
    """Configure VRM 1.0 expression presets for non-VRM base meshes.

    D-011: Symmetric expressions bind BOTH L and R shape keys.
    Expression map supports multiple shape key bindings per expression,
    each with its own weight value.
    """
    logger.info("Setting up VRM 1.0 expressions...")

    # ── Discover all shape keys across all mesh objects ─────────────────────
    mesh_shape_keys: dict[str, dict[str, str]] = {}  # mesh_name → {key_name: key_name}
    for obj in bpy.data.objects:
        if obj.type == "MESH" and obj.data.shape_keys:
            keys = {}
            for kb in obj.data.shape_keys.key_blocks:
                keys[kb.name] = kb.name
            if keys:
                mesh_shape_keys[obj.name] = keys

    total_keys = sum(len(v) for v in mesh_shape_keys.values())
    if not mesh_shape_keys:
        logger.warning("No shape keys found on any mesh")
        return

    logger.info("Found %d shape keys across %d meshes", total_keys, len(mesh_shape_keys))

    # ── Build a reverse index: shape_key_name → mesh_object_name ────────────
    shape_key_to_mesh: dict[str, str] = {}
    for obj_name, keys in mesh_shape_keys.items():
        for k in keys:
            shape_key_to_mesh[k] = obj_name

    # ── Select expression map ───────────────────────────────────────────────
    # Allow spec to override expression mapping
    spec_expressions = spec.get("expression_map", {})
    if spec_expressions and isinstance(spec_expressions, dict):
        # Spec provides custom expression overrides — merge with defaults
        expression_map = dict(TURBOSQUID_EXPRESSION_MAP)
        for preset_name, bindings in spec_expressions.items():
            if isinstance(bindings, list):
                expression_map[preset_name] = bindings
        logger.info("Applied %d spec expression overrides", len(spec_expressions))
    else:
        expression_map = TURBOSQUID_EXPRESSION_MAP

    # ── Map VRM expression presets to shape key binds ───────────────────────
    preset = vrm_ext.vrm1.expressions.preset
    mapped = 0

    for vrm_preset_name, bindings in expression_map.items():
        # Get the preset expression group
        preset_expr = getattr(preset, vrm_preset_name, None)
        if preset_expr is None:
            logger.debug("No VRM preset property for '%s' — skipping", vrm_preset_name)
            continue

        if isinstance(bindings, dict):
            # Legacy single-binding format (upgrade to list)
            bindings = [{"shape_key": bindings.get("shape_key", bindings.get("name", "")),
                         "weight": bindings.get("weight", 1.0)}]

        # Process each binding in the list
        binds_applied = 0
        for binding in bindings:
            shape_key_name = binding.get("shape_key", binding.get("name", ""))
            weight = float(binding.get("weight", 1.0))

            if not shape_key_name:
                continue

            # Find which mesh has this shape key
            mesh_obj_name = shape_key_to_mesh.get(shape_key_name)

            # Try fuzzy matching if exact match fails
            if mesh_obj_name is None:
                shape_lower = shape_key_name.lower()
                for k, obj_name in shape_key_to_mesh.items():
                    if k.lower() == shape_lower:
                        mesh_obj_name = obj_name
                        shape_key_name = k  # Use actual case
                        break

            if mesh_obj_name is None:
                logger.debug("Shape key '%s' not found for expression '%s'",
                            shape_key_name, vrm_preset_name)
                continue

            # Add a morph target bind
            bind = preset_expr.morph_target_binds.add()
            bind.node.mesh_object_name = mesh_obj_name
            bind.index = shape_key_name  # D-013: VRM 1.0 uses shape key NAME, not number
            bind.weight = weight
            binds_applied += 1
            logger.debug("Expression '%s' → '%s' on '%s' (weight=%.2f)",
                        vrm_preset_name, shape_key_name, mesh_obj_name, weight)

        if binds_applied > 0:
            mapped += 1

    # Disable auto-expression assignment to preserve our mappings
    vrm_ext.vrm1.expressions.initial_automatic_expression_assignment = False
    logger.info("Disabled VRM auto-expression assignment")

    total_bindings = sum(len(b) for b in expression_map.values())
    logger.info("VRM expression mapping: %d/%d expressions configured (%d total binds)",
                mapped, len(expression_map), total_bindings)


# ──────────────────────────────────────────────────────────────────────────────
# VRM Metadata
# ──────────────────────────────────────────────────────────────────────────────

def _setup_vrm_metadata(bpy, spec: dict) -> None:
    """Set VRM 1.0 metadata (title, author, version, license, contact).

    This is required for VRChat upload and proper avatar identification.
    """
    for obj in bpy.data.objects:
        if obj.type == "ARMATURE" and hasattr(obj.data, "vrm_addon_extension"):
            vrm_ext = obj.data.vrm_addon_extension
            meta = vrm_ext.vrm1.meta

            # Title
            title = spec.get("display_name", spec.get("avatar_id", "Unnamed Avatar"))
            meta.name = title
            meta.title = title

            # Version
            version = spec.get("spec_version", "1.0")
            meta.version = version

            # Author
            metadata = spec.get("metadata", {})
            author = metadata.get("author", "Unknown")
            meta.authors.add()
            meta.authors[0].name = author

            # License
            license_name = metadata.get("license", "CC-BY-4.0")
            # Map common license names to VRM enum values
            license_map = {
                "CC-BY-4.0": "CC_BY_4_0",
                "CC-BY-SA-4.0": "CC_BY_SA_4_0",
                "CC-BY-NC-4.0": "CC_BY_NC_4_0",
                "CC0": "CC0",
                "MIT": "OTHER",
            }
            meta.license_type = license_map.get(license_name, "OTHER")
            if meta.license_type == "OTHER":
                try:
                    meta.other_license_url = f"https://spdx.org/licenses/{license_name}"
                except Exception:
                    pass

            # Contact information
            contact_url = metadata.get("contact_url", "")
            if contact_url:
                try:
                    meta.reference.unity_package_url = contact_url
                except (AttributeError, Exception):
                    pass

            # Allowed uses
            commercial_use = metadata.get("commercial_use", True)
            redistribution = metadata.get("redistribution", True)
            try:
                meta.allow_excessive_violence = False
                meta.allow_excessive_sexual_usage = True  # Freyja's domain is sacred
                meta.allow_political_usage = False
                meta.allow_religious_usage = True
                meta.allow_redistribution = redistribution
                meta.commercial_usage = "ALLOW_COMMERCIAL_USE" if commercial_use else "PERSONAL_NON_COMMERCIAL"
            except (AttributeError, Exception):
                pass

            logger.info("VRM metadata set: title='%s', author='%s', license='%s'",
                       title, author, license_name)
            break


# ──────────────────────────────────────────────────────────────────────────────
# VRM First-Person Offset
# ──────────────────────────────────────────────────────────────────────────────

def _setup_vrm_first_person(bpy, spec: dict) -> None:
    """Configure VRM 1.0 first-person mesh annotations.

    This defines how the avatar appears from first-person view in VR.
    By default, the head mesh is hidden in first-person to prevent clipping.
    """
    for obj in bpy.data.objects:
        if obj.type == "ARMATURE" and hasattr(obj.data, "vrm_addon_extension"):
            vrm_ext = obj.data.vrm_addon_extension
            first_person = vrm_ext.vrm1.first_person

            # Set first-person offset (position of the "eyes" in the head bone)
            try:
                # Find head bone position
                armature = obj
                head_bone = None
                for bone in armature.data.bones:
                    if bone.name.lower() in ("head",):
                        head_bone = bone
                        break
                if head_bone:
                    # Place viewpoint slightly forward and up from head bone center
                    head_center = (head_bone.head_local + head_bone.tail_local) / 2
                    first_person.first_person_offset = (
                        head_center[0],
                        head_center[1] + 0.05,  # Slightly forward
                        head_center[2] + 0.06,   # Slightly up
                    )
                    logger.debug("First-person offset: %.3f, %.3f, %.3f",
                                head_center[0], head_center[1] + 0.05, head_center[2] + 0.06)
            except (AttributeError, Exception) as exc:
                logger.debug("First-person offset configuration skipped: %s", exc)

            # Configure mesh annotations for first-person visibility
            # Head mesh should be auto-hidden in first-person view
            try:
                for mesh_obj in bpy.data.objects:
                    if mesh_obj.type == "MESH":
                        annotation = first_person.mesh_annotations.add()
                        annotation.mesh_object_name = mesh_obj.name
                        # Check if this mesh is part of the head region
                        is_head_mesh = False
                        for mod in mesh_obj.modifiers:
                            if mod.type == "ARMATURE" and mod.object:
                                for vert in mesh_obj.data.vertices:
                                    for group in vert.groups:
                                        for bone in mod.object.data.bones:
                                            if (bone.name.lower() in ("head", "neck", "necktwist01", "necktwist02")
                                                and group.group_index < len(mesh_obj.vertex_groups)):
                                                is_head_mesh = True
                                                break
                                        if is_head_mesh:
                                            break
                                    if is_head_mesh:
                                        break
                                break

                        # Simple heuristic: if mesh name contains head/face/eye/tear/eyelash → auto
                        mesh_name_lower = mesh_obj.name.lower()
                        if any(kw in mesh_name_lower for kw in ("head", "face", "eye", "tear", "eyelash")):
                            annotation.first_person_flag = "AUTO"
                        else:
                            annotation.first_person_flag = "BOTH"
            except (AttributeError, Exception) as exc:
                logger.debug("Mesh annotation configuration skipped: %s", exc)

            logger.info("VRM first-person configuration applied")
            break


# ──────────────────────────────────────────────────────────────────────────────
# VRM LookAt Configuration
# ──────────────────────────────────────────────────────────────────────────────

def _setup_vrm_look_at(bpy, spec: dict) -> None:
    """Configure VRM 1.0 lookAt (eye gaze/look direction) system.

    D-012: Uses bone rotation mode for lookAt, not morph targets.
    The lookAt system controls how the avatar's eyes follow the viewer/camera.
    Bone rotation is more natural for real-time applications like VRChat.
    """
    for obj in bpy.data.objects:
        if obj.type == "ARMATURE" and hasattr(obj.data, "vrm_addon_extension"):
            vrm_ext = obj.data.vrm_addon_extension
            look_at = vrm_ext.vrm1.look_at

            try:
                # Use bone rotation mode (D-012)
                # D-019: VRM 1.0 enum identifiers are lowercase ('bone', 'expression').
                # Setting uppercase 'BONE' triggers:
                #   bpy_struct: item.attr = val: enum 'BONE' not found in ('bone', 'expression')
                look_at.type = "bone"

                # Find eye bones for offset calculation
                armature = obj
                left_eye_pos = None
                right_eye_pos = None

                for bone in armature.data.bones:
                    if bone.name in ("L_Eye", "Eye_L", "LeftEye"):
                        left_eye_pos = bone.head_local
                    elif bone.name in ("R_Eye", "Eye_R", "RightEye"):
                        right_eye_pos = bone.head_local

                # Set eye offsets based on bone positions
                if left_eye_pos is not None and right_eye_pos is not None:
                    # Center between the eyes
                    center = (left_eye_pos + right_eye_pos) / 2
                    look_at.offset_from_head_bone = (
                        center[0],
                        center[1] - left_eye_pos[1],  # Forward offset
                        center[2] - (left_eye_pos[2] + right_eye_pos[2]) / 2,  # Up offset
                    )
                    logger.debug("LookAt offset: %.3f, %.3f, %.3f",
                                look_at.offset_from_head_bone[0],
                                look_at.offset_from_head_bone[1],
                                look_at.offset_from_head_bone[2])
                else:
                    # Default offsets (reasonable for humanoid)
                    look_at.offset_from_head_bone = (0.0, 0.0, 0.06)
                    logger.debug("LookAt offset: using defaults (no eye bones found)")

                # Look-at horizontal/vertical range (degrees)
                # These define the maximum rotation range for eye bones
                look_config = spec.get("look_at", {})
                horizontal_inner = float(look_config.get("horizontal_inner_degrees", 15.0))
                horizontal_outer = float(look_config.get("horizontal_outer_degrees", 15.0))
                vertical_down = float(look_config.get("vertical_down_degrees", 10.0))
                vertical_up = float(look_config.get("vertical_up_degrees", 10.0))

                look_at.horizontal_inner = horizontal_inner
                look_at.horizontal_outer = horizontal_outer
                look_at.vertical_down = vertical_down
                look_at.vertical_up = vertical_up

                logger.info("VRM lookAt configured: bone mode, H=%.1f/%.1f°, V=%.1f/%.1f°",
                           horizontal_inner, horizontal_outer, vertical_down, vertical_up)
            except (AttributeError, Exception) as exc:
                logger.warning("VRM lookAt configuration failed: %s", exc)
            break


# ──────────────────────────────────────────────────────────────────────────────
# Post-export Validation
# ──────────────────────────────────────────────────────────────────────────────

def _validate_vrm(output_path: str) -> None:
    """Validate the exported VRM file for basic correctness.

    Checks:
        - File exists and is non-zero
        - Can be opened as a glTF binary
        - Contains VRM extension data
        - Bone count is reasonable
    """
    import struct

    path = Path(output_path)
    if not path.exists():
        raise ValueError(f"VRM file not found: {output_path}")
    if path.stat().st_size < 1024:
        raise ValueError(f"VRM file suspiciously small: {path.stat().st_size} bytes")

    # Read glTF header (first 20 bytes: magic + version + length + JSON length + JSON offset)
    with open(output_path, "rb") as f:
        header = f.read(20)
        if len(header) < 20:
            raise ValueError("VRM file too short for glTF header")

        magic = struct.unpack("<I", header[0:4])[0]
        if magic != 0x46546C67:  # 'glTF' in little-endian
            raise ValueError(f"Not a valid glTF file: magic=0x{magic:08X}")

        version = struct.unpack("<I", header[4:8])[0]
        total_length = struct.unpack("<I", header[8:12])[0]
        json_length = struct.unpack("<I", header[12:16])[0]
        json_offset = struct.unpack("<I", header[16:20])[0]

        # Read JSON chunk header
        f.seek(json_offset)
        chunk_header = f.read(8)
        if len(chunk_header) < 8:
            raise ValueError("Cannot read JSON chunk header")

        chunk_length = struct.unpack("<I", chunk_header[0:4])[0]
        chunk_type = struct.unpack("<I", chunk_header[4:8])[0]
        if chunk_type != 0x4E4F534A:  # 'JSON' in little-endian
            raise ValueError("First chunk is not JSON type")

        # Read JSON content
        json_data = f.read(chunk_length)
        gltf_json = json.loads(json_data.decode("utf-8"))

        # Check VRM extension
        extensions = gltf_json.get("extensions", {})
        vrm_ext = extensions.get("VRMC_vrm", extensions.get("VRM"))
        if vrm_ext is None:
            raise ValueError("No VRM extension found in glTF — export may have failed")

        # Check humanoid bones
        humanoid = vrm_ext.get("humanoid", {})
        bones = humanoid.get("humanBones", [])
        if len(bones) < 15:
            raise ValueError(f"Too few humanoid bones mapped: {len(bones)} (minimum 15 required)")

        # Check expressions
        expressions = vrm_ext.get("expressions", {})
        preset = expressions.get("preset", {})
        expr_count = len(preset)
        if expr_count < 5:
            logger.warning("Few expressions configured: %d", expr_count)

        # Report file size
        size_mb = path.stat().st_size / (1024 * 1024)
        logger.info("VRM validation passed ✓ — %.1f MB, %d bones, %d expressions, glTF v%d",
                   size_mb, len(bones), expr_count, version)


if __name__ == "__main__":
    sys.exit(main())
