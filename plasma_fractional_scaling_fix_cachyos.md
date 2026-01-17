this is a very common issue and the simple fix is adding this like
set -gx QT_SCALE_FACTOR_ROUNDING_POLICY RoundPreferFloor
in ~/.config/fish/config.fish config file.
and then after reboot you shouldn't be seeing any blured artifacts.
