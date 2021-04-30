-- ===================== --
-- UniverseCanSmoosh
-- ===================== --
-- Version 1.0
-- Created by Charlie Hall and Ellie Garnett
-- https://www.github.com/charlie9830
-- Last Updated April 2021
--
-- Source Code available at:
-- https://github.com/Charlie9830/UniverseCanSmoosh
--
--
-- DESCRIPTION --
-- =========== --
-- This plugin will determine the number of free addresses on a given universe and the largest currently free range,
-- allowing for additional fixtures to be added if addresses are smooshed up.
-- 
--
-- IMPORTANT --
-- ========= --
-- Due to how MA calculates parameters for FixtureTypes with 'Virtual Channels', this plugin cannot correctly count parameters
-- for FixtureTypes with 'Virtual Channels'. If you have a number of FixtureTypes with 'Virtual Channels' the parameter count
-- reported by this plugin will be LESS then what the actual parameter count is.
--
-- LICENSES --
-- ======== --
-- ParameterCounter
-- MIT License
-- Copyright (c) 2021 Charlie Hall and Ellie Garnett
-- Permission is hereby granted, free of charge, to any person obtaining a copy
-- of this software and associated documentation files (the "Software"), to deal
-- in the Software without restriction, including without limitation the rights
-- to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
-- copies of the Software, and to permit persons to whom the Software is
-- furnished to do so, subject to the following conditions:
-- The above copyright notice and this permission notice shall be included in all
-- copies or substantial portions of the Software.
-- THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
-- IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
-- FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
-- AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
-- LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
-- OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
-- SOFTWARE.
-- CONFIG
-- PROGRESS HANDLES
local paramCountProgressHandle

local function clamp(low, n, high)
    return math.min(math.max(n, low), high)
end

local function validateUniverseNumber(input)
    local asNumber = tonumber(input)

    if asNumber == nil then
        return false, 1
    end

    return true, clamp(1, math.floor(asNumber), 128);
end

local function calculateFreeAddresses(universeNumber, progressHandle)
    local freeAddrCount = 0;
    local largestRange = {
        startAddr = 1,
        endAddr = 1,
        size = 0
    }

    local currentRange = {
        startAddr = 1,
        endAddr = 1,
        size = 0
    }

    gma.gui.progress.setrange(progressHandle, 1, 512);
    for addr = 1, 512 do
        local handle = gma.show.getobj.handle("DMX " .. universeNumber .. '.' .. addr);
        local fixId = gma.show.property.get(handle, "FixId");
        local chaId = gma.show.property.get(handle, "ChaId");

        if fixId == "" and chaId == "" then
            -- Free Address
            freeAddrCount = freeAddrCount + 1

            if currentRange.endAddr == addr - 1 then
                -- currentRange is Contiguous so update it.
                currentRange.endAddr = addr;
                currentRange.size = currentRange.size + 1
            else
                -- Current Range is not contiguous. This will become the begining of a new Range.
                -- Update Largest range if this is the new champion and then reset the current Range to our current Address.
                if (currentRange.size > largestRange.size) then
                    largestRange.startAddr = currentRange.startAddr;
                    largestRange.endAddr = currentRange.endAddr;
                    largestRange.size = currentRange.size;
                end

                -- Reseting Current Range.
                currentRange.startAddr = addr;
                currentRange.endAddr = addr;
                currentRange.size = 0;
            end

            if addr == 512 and currentRange.size > largestRange.size then
                -- End of Universe. Check if we have a new largest Range and update accordingly.
                largestRange.startAddr = currentRange.startAddr;
                largestRange.endAddr = currentRange.endAddr;
                largestRange.size = currentRange.size;
            end

        else
            -- Patched Address
            if (currentRange.size > largestRange.size) then
                -- Update Largest Range if we have a new Largest Range.
                largestRange.startAddr = currentRange.startAddr;
                largestRange.endAddr = currentRange.endAddr;
                largestRange.size = currentRange.size;
            end

            -- Reset current Range
            currentRange.startAddr = 1;
            currentRange.endAddr = 1;
            currentRange.size = 0;
        end

        if (math.fmod(addr, 5) == 0) then
            gma.gui.progress.set(progressHandle, addr)
        end

    end

    gma.gui.progress.stop(progressHandle);

    return freeAddrCount, largestRange;
end

local function askForUniverseNumber()
    local result = gma.textinput("Enter Universe Number");

    local isValid, universeNumber = validateUniverseNumber(result);

    return isValid, universeNumber;
end

local function Main()

    local isValid, universeNumber = askForUniverseNumber();

    if isValid == false then
        gma.gui.msgbox("Error", "Invalid universe Number");
        return;
    end

    local freeAddr, largestRange = calculateFreeAddresses(universeNumber)

    local message = ''
    if freeAddr == 0 then
        message = 'There are no available addresses on Universe ' .. universeNumber;
    elseif freeAddr == 1 then
        message = "Address " .. freeAddr .. " is the ONLY available address on Universe " .. universeNumber;
    else
        message = freeAddr .. ' addresses available on Universe ' .. universeNumber ..
                      '.\nThe largest continuous range being ' .. largestRange.size + 1 .. ' addresses in length ' ..
                      'from  ' .. universeNumber .. '/' .. largestRange.startAddr .. '  to  ' .. universeNumber .. '/' ..
                      largestRange.endAddr;
    end

    gma.gui.msgbox("Done", message)
end

local function Cleanup()
    gma.gui.progress.stop(paramCountProgressHandle);
end

return Main, Cleanup
