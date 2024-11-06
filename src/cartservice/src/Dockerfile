# Copyright 2021 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#FROM mcr.microsoft.com/dotnet/sdk:9.0@sha256:5d337a63df98dc6b846084e12d903bfdf17539ee57032721d619ae444e7482b7 AS build
FROM mcr.microsoft.com/dotnet/sdk:8.0.403-noble-arm64v8 AS build
WORKDIR /app
COPY . .
RUN dotnet restore cartservice.csproj
RUN dotnet build "./cartservice.csproj" -c Debug -r linux-arm64 -o /out

FROM build AS publish
RUN dotnet publish cartservice.csproj -c Debug -r linux-arm64 -o /out --self-contained false

# Building final image used in running container
#FROM mcr.microsoft.com/dotnet/aspnet:9.0@sha256:536ba20cfb467dae1307be48a5a90d6bc3b1a609c9ecbf49bb22b0b4211f897e AS final
FROM mcr.microsoft.com/dotnet/sdk:8.0.403-noble-arm64v8 AS final
# Installing procps on the container to enable debugging of .NET Core
RUN apt-get update \
    && apt-get install -y unzip procps  wget curl

# Download and install the Datadog Tracer
#ENV TRACER_VERSION=3.3.1
RUN mkdir -p /opt/datadog \
    && mkdir -p /var/log/datadog \
    && TRACER_VERSION=$(curl -s https://api.github.com/repos/DataDog/dd-trace-dotnet/releases/latest | grep tag_name | cut -d '"' -f 4 | cut -c2-) \
    && curl -LO https://github.com/DataDog/dd-trace-dotnet/releases/download/v${TRACER_VERSION}/datadog-dotnet-apm_${TRACER_VERSION}_arm64.deb \
    && dpkg -i ./datadog-dotnet-apm_${TRACER_VERSION}_arm64.deb \
    && /opt/datadog/createLogPath.sh \
    && rm ./datadog-dotnet-apm_${TRACER_VERSION}_arm64.deb

WORKDIR /app
COPY --from=publish /out .
ENV ASPNETCORE_HTTP_PORTS=7070
#Enable the tracer
ENV CORECLR_ENABLE_PROFILING=1
ENV CORECLR_PROFILER={846F5F1C-F9AE-4B07-969E-05C26BC060D8}
ENV CORECLR_PROFILER_PATH=/opt/datadog/linux-arm64/Datadog.Trace.ClrProfiler.Native.so
ENV DD_DOTNET_TRACER_HOME=/opt/datadog
ENV DD_INTEGRATIONS=/opt/datadog/integrations.json
ENV LD_PRELOAD=/opt/datadog/linux-arm64/Datadog.Linux.ApiWrapper.x64.so
ENV DD_PROFILING_ENABLED=1
ENV DD_TRACE_DEBUG=0
USER 1000
ENTRYPOINT ["dotnet", "cartservice.dll"]
